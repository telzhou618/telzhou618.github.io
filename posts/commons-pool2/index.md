# Commons-pool2 应用与实战

---
把创建耗时的对象提前创建好，放在池中管理，需要时直接取，用完归还，可以做对象复用，避免用时创建慢且提高系统的效率。
比如数据库连接、网络连接、大对象创建等都可以池化处理。



![image-20210811195620000](https://raw.gitmirror.com/telzhou618/images/main/img/image-20210811195620000.png)

## commons-pool的优点

对象复用，减少创建开销。对于数据库的连接池可以减少创建连接带来的开销，连接用完归还可以给其他线程复用。

## 三个核心对象

### GenericObjectPool 

连接池对象，继承自BaseGenericObjectPool  用双端阻塞队列LinkedBlockingDeque保存空闲链接，用ConcurrentHashMap保存所有的链接。

构造方法

```java
public GenericKeyedObjectPool(KeyedPooledObjectFactory&lt;K, T&gt; factory, GenericKeyedObjectPoolConfig&lt;T&gt; config) {
  //...
}
```

GenericObjectPoolConfig和PooledObjectFactory是创建对象池所必须的两个对象，稍后详细说明。

重要方法

- borrowObject() 从连接池中取对象。

- returnObject() 归还对象。

### PooledObjectFactory 

生产对象的工厂，有如下一些方法：

- makeObject() 生产一个对象
  - 连接池初始化,驱逐过期连接后小于最小连接数，所有连接被占用且总连接数小于最大连接时调用。
- destroyObject() 销毁一个对象
- validateObject() 校验一个对象
- activateObject() 重新激活一个对象
- passivateObject() 钝化一个对象

没有特别要求可用PooledObjectFactory 的子类 BasePooledObjectFactory作为生产对象的工厂更简洁。

- create() 生产对象的逻辑

- wrap() 将对象包装成 PooledObject，必须支持多线程。

### GenericObjectPoolConfig

连接池配置对象，继承自BaseObjectPoolConfig 。

常用参数：

maxTotal：对象池中管理的最多对象个数。默认值是8。
maxIdle：对象池中最大的空闲对象个数。默认值是8。
minIdle：对象池中最小的空闲对象个数。默认值是0。

## 实例代码

创建一个简单的连接对象，模拟数据库连接对象。

```java
   /**
     * 连接对象
     */
    @Getter
    @Setter
    @ToString
    static class Connection {
        private Integer id;

        public Connection(Integer id) {
            this.id = id;
        }
    }
```

生产连接对象的工厂,需要继承 BasePooledObjectFactory 类

```java
 /**
     * 对象生产工厂
     */
    static class ConnectionObjectFactory extends BasePooledObjectFactory&lt;Connection&gt; {

        private AtomicInteger atomicInteger = new AtomicInteger(1);

        @Override
        public Connection create() throws Exception {
            // 模拟连接对象的创建过程
            return new Connection(atomicInteger.getAndIncrement());
        }

        @Override
        public PooledObject&lt;Connection&gt; wrap(Connection connection) {
            // 将连接包装成PooledObject对象，要线程安全
            return new DefaultPooledObject&lt;&gt;(connection);
        }
    }
```

自定义配置对象，继承GenericObjectPoolConfig，如不需要特殊配置可直接用GenericObjectPoolConfig。

```java
    /**
     * 连接池配置信息
     */
    static class ConnectionPoolConfig extends GenericObjectPoolConfig {
        public ConnectionPoolConfig() {
            setTestWhileIdle(true);
            setMinEvictableIdleTimeMillis(60000);
            setTimeBetweenEvictionRunsMillis(30000);
            setNumTestsPerEvictionRun(-1);
        }
    }
```

使用

```java
    public static void main(String[] args) throws Exception {

        // 实例化对象生产工厂
        ConnectionObjectFactory factory = new ConnectionObjectFactory();
        // 实例化连接池配置信息
        ConnectionPoolConfig config = new ConnectionPoolConfig();
        config.setMaxTotal(5);
        config.setMaxWaitMillis(1000);
        // 实例化连接池
        GenericObjectPool&lt;Connection&gt; pool = new GenericObjectPool&lt;&gt;(factory, config);

        for (int i = 0; i &lt; 10; i&#43;&#43;) {
            // 获取连接
            Connection connection = pool.borrowObject();
            System.out.println(connection);
            // 归还连接
            //  pool.returnObject(connection);
        }

    }
```

先注释归还对象的方法 pool.returnObject(connection)，执行结果如下，只能取到5个连接对象，因为设置的最大连接数是5。

![image-20210729144629504](https://raw.gitmirror.com/telzhou618/images/main/img/image-20210729144629504.png)

在打开  pool.returnObject(connection) 代码，每次获取的第一个连接，说明归还了对象可以接着用，实现连接对象复用。

![image-20210729144822189](https://raw.gitmirror.com/telzhou618/images/main/img/image-20210729144822189.png)

## Redis客户端jedis的的连接池典型应用

### JedisPoolConfig

配置类源码, 只修改了默认的几个参数，其余继承GenericObjectPoolConfig

```java
public class JedisPoolConfig extends GenericObjectPoolConfig {
  public JedisPoolConfig() {
    // defaults to make your life with connection pool easier :)
    setTestWhileIdle(true);
    setMinEvictableIdleTimeMillis(60000);
    setTimeBetweenEvictionRunsMillis(30000);
    setNumTestsPerEvictionRun(-1);
  }
}
```

### JedisFactory

生成jedis对象的工厂，继承commons-pool2的 PooledObjectFactory，重点实现了makeObject()生产jedis对象。

```java
class JedisFactory implements PooledObjectFactory&lt;Jedis&gt; {
    @Override
    public PooledObject&lt;Jedis&gt; makeObject() throws Exception {
      final HostAndPort hostAndPort = this.hostAndPort.get();
      final Jedis jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort(), connectionTimeout,
          soTimeout, ssl, sslSocketFactory, sslParameters, hostnameVerifier);

      try {
        jedis.connect();
        if (null != this.password) {
          jedis.auth(this.password);
        }
        if (database != 0) {
          jedis.select(database);
        }
        if (clientName != null) {
          jedis.clientSetname(clientName);
        }
      } catch (JedisException je) {
        jedis.close();
        throw je;
      }

      return new DefaultPooledObject&lt;Jedis&gt;(jedis);

    }
}
```

### JedisPool

实例化jedis连接池，初始化连接对象。

```java
public class JedisPool extends Pool&lt;Jedis&gt; {
   protected GenericObjectPool&lt;T&gt; internalPool;
  // 实例化连接池 参数省略...
  public JedisPool(hostname,port,...){
    this.internalPool = new GenericObjectPool&lt;Jedis&gt;(new JedisFactory(host,
          Protocol.DEFAULT_PORT, Protocol.DEFAULT_TIMEOUT, Protocol.DEFAULT_TIMEOUT, null,
          Protocol.DEFAULT_DATABASE, null, false, null, null, null), new GenericObjectPoolConfig());
    }
  
  @Override
  public Jedis getResource() {
    //获取jedis对象
  }
  @Override
  public void returnResource(final Jedis resource) {
    //归还jedis对象
  }
}
```

最终调用GenericObjectPool作为对象池管理jedis对象，传入JedisFactory工厂和默认的配置对象GenericObjectPoolConfig，配置类可以使用redis自定义的JedisPoolConfig对象。

Jedis客户端通过依赖commons-pool2实现连接池，使得Jedis本身只关注自身和Redis服务的交互上，不需要关系连接怎么创建、销毁等。

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/commons-pool2/  

