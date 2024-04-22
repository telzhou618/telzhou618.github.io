# Redis 或 Zookeeper 实现分布式锁

---
分布式锁是为了解决跨进程、跨服务应用在高并发场景下造成线程不安全的问题的一种同步加锁的技术方案，在互联网公司有广泛的使用场景，如秒杀项目减库存场景，分布式锁可以保证所有参与者按顺序排队访问某一资源，谁排在最前面谁就有资格获得资源的使用权，排在后面的线程必须等到持有锁的线程释放锁才有可能获取资源使用权，后面的线程要么等待要么放弃。

分布式锁是缺点：造型系统 **吞吐量、可用性** 下降。


## Redis 实现分布式锁

- 需要用到redisson包,新建spring-boot项目，导入redisson包
```xml
 &lt;dependency&gt;
    &lt;groupId&gt;org.redisson&lt;/groupId&gt;
    &lt;artifactId&gt;redisson-spring-boot-starter&lt;/artifactId&gt;
    &lt;version&gt;3.15.6&lt;/version&gt;
&lt;/dependency&gt;
```

- 配置Redis连接信息
```yaml
spring:
  redis:
    database: 0
    host: 127.0.0.1
    port: 6379
```

- 锁的具体使用代码
```java
@RestController
@AllArgsConstructor
public class DistributedLockRedissonController {

    private final RedissonClient redissonClient;
    private final StringRedisTemplate stringRedisTemplate;

    /**
     * redisson 分布式锁
     */
    @RequestMapping(&#34;/redisson/doReduceStack&#34;)
    public String doReduceStack() {
        // 检查库存
        int store = Integer.parseInt(stringRedisTemplate.opsForValue().get(&#34;store&#34;));
        if (store &lt;= 0) {
            throw new RuntimeException(&#34;库存不足&#34;);
        }
        RLock lock = redissonClient.getLock(&#34;lock&#34;);
        try {
            lock.lock();
            // 双重检查
            store = Integer.parseInt(stringRedisTemplate.opsForValue().get(&#34;store&#34;));
            if (store &lt;= 0) {
                throw new RuntimeException(&#34;库存不足&#34;);
            }
            // 减库存
            stringRedisTemplate.opsForValue().set(&#34;store&#34;, String.valueOf(store - 1));
            // 生成订单
            System.out.println(&#34;下单成功&#34;);
            return &#34;下单成功&#34;;
        } finally {
            if (lock.isLocked() &amp;&amp; lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```
- 实现原理

待完善


## ZK 实现分布式锁

- 需要用到curator包，新建spring-boot项目，导入curator包
```xml
&lt;dependency&gt;
    &lt;groupId&gt;org.apache.curator&lt;/groupId&gt;
    &lt;artifactId&gt;curator-framework&lt;/artifactId&gt;
    &lt;version&gt;5.1.0&lt;/version&gt;
&lt;/dependency&gt;
```


- 配置zookeeper连接信息
```yaml
spring:
      zookeeper:
        address: 127.0.0.1:2181
```

- 注册CuratorFramework客户端
```java
@Configuration
public class CommonConfig {

    @Value(&#34;${spring.zookeeper.address}&#34;)
    private String zookeeperAddress;

    @Bean
    @ConditionalOnMissingBean({CuratorFramework.class})
    public CuratorFramework curatorFramework() {
        CuratorFramework curatorFramework = CuratorFrameworkFactory.newClient(zookeeperAddress, new RetryNTimes(5, 1000));
        curatorFramework.start();
        return curatorFramework;
    }
}
```


- zookeeper分布式锁使用
```java
@RestController
@AllArgsConstructor
public class DistributedLockZookeeperController {

    private CuratorFramework curatorFramework;

    /**
     * zookeeper 分布式锁
     */
    @RequestMapping(&#34;/zookeeper/get-lock&#34;)
    public String doReduceStack() throws Exception {
        InterProcessMutex lock = new InterProcessMutex(curatorFramework, &#34;/zookeeper/lockId&#34;);

        // zookeeper 加锁的两种方式

        // lock.acquire()
        // 尝试加锁，如果加锁失败，会一致的等到加锁成功。

        // lock.acquire(3, TimeUnit.SECONDS)
        // 尝试加锁，如果加锁失败会在3秒内不断获取所，如果3秒内获取锁失败，则抛异常

        if (lock.acquire(3, TimeUnit.SECONDS)) {
            try {
                // 执行业务
                System.out.println(&#34;获得锁成功，执行业务逻辑！&#34;);
                TimeUnit.SECONDS.sleep(2);
                return &#34;success&#34;;
            } finally {
                if (lock.isOwnedByCurrentThread()) {
                    lock.release();
                }
            }
        } else {
            throw new RuntimeException(&#34;操作过于频繁&#34;);
        }
    }

}
```
- 实现原理

待完善

## 分布式锁总结


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/distributed-lock/  

