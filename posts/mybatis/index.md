# MyBatis 源码解析

一款轻量级的ORM框架。

![image-20210730150741225](https://raw.githubusercontent.com/telzhou618/images/main/img/image-20210730150741225.png)

## MyBatis 的优点

- 封装底层乏味的JDBC操作，让开发中更关注业务。
- SQL语句写在XML里和代码分离，便于维护，低耦合。

## MyBatis 的缺点

- 相比全自动的hibernat, SQL编写工作量大。
- 对数据库SQL依赖比较强，移植性差。

## MyBatis 核心组件

### Configuration

配置类，MyBatis启动是会解析全局配置文件全局配置文件 mybatis-config.xml及所有的 XXXMapper.xml文件,解析结果存入Configuration对象中，该对象是单例的，存在会话的上下文，贯穿这个MyBatis的执行过程。

### SqlSession

SQL执行的顶层接口，定义了和数据库交互的所有方法，有CRUD及开始事务、提交事务、回滚事务等，有一个默认的实现类 DefaultSqlSession，持有Configuration和Executor对象，SqlSession本身没有做多少有用的事，具体的SQL语句执行委托给了Executor执行。

### Executor

执行器，实现了SqlSession定义的SQL操作方法，实现了MyBatis的一级缓存，具体的SQL执行委托给它的下一级 StatementHandler。

### StatementHandler

真正和数据交互的对象，实现了执行SQL语句，自身持有ParameterHandler和ResultSetHandler对象。

### ParameterHandler

负责参数解析。

### ResultSetHandler

负责处理SQL执行结果。

以上是MyBtais重要的几个组成部件，下面分析一下具体的源码执行流程。

## MyBatis 源码分析

### 从Configuration配置解析开始

实例如下：

```java
public static void main(String[] args) throws IOException {
  String resource = &#34;mybatis-config.xml&#34;;
  InputStream inputStream = Resources.getResourceAsStream(resource);
  SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
  SqlSession session = sqlSessionFactory.openSession();
  Blog blog = session.selectOne(&#34;com.example.mapper.BlogMapper.selectBlog&#34;, 1);
  System.out.println(blog);
}
```

从 new SqlSessionFactoryBuilder().build(inputStream) 入手解读配置解析

```java
  public SqlSessionFactory build(InputStream inputStream) {
     return build(inputStream, null, null);
  }
  
  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException(&#34;Error building SqlSession.&#34;, e);
    } finally {
      //...
    }
  }

  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

关键方法 parser.parse(), 这里完成了所以配置文件的解析。

```java
// 解析配置
public Configuration parse() {
  //... 省略
  parseConfiguration(parser.evalNode(&#34;/configuration&#34;));
  return configuration;
}

// 解析配置
private void parseConfiguration(XNode root) {
  try {
    //分步骤解析
    //issue #117 read properties first
    //1.properties
    propertiesElement(root.evalNode(&#34;properties&#34;));
    //2.类型别名
    typeAliasesElement(root.evalNode(&#34;typeAliases&#34;));
    //3.插件
    pluginElement(root.evalNode(&#34;plugins&#34;));
    //4.对象工厂
    objectFactoryElement(root.evalNode(&#34;objectFactory&#34;));
    //5.对象包装工厂
    objectWrapperFactoryElement(root.evalNode(&#34;objectWrapperFactory&#34;));
    //6.设置
    settingsElement(root.evalNode(&#34;settings&#34;));
    // read it after objectFactory and objectWrapperFactory issue #631
    //7.环境
    environmentsElement(root.evalNode(&#34;environments&#34;));
    //8.databaseIdProvider
    databaseIdProviderElement(root.evalNode(&#34;databaseIdProvider&#34;));
    //9.类型处理器
    typeHandlerElement(root.evalNode(&#34;typeHandlers&#34;));
    //10.映射器
    mapperElement(root.evalNode(&#34;mappers&#34;));
  } catch (Exception e) {
    throw new BuilderException(&#34;Error parsing SQL Mapper Configuration. Cause: &#34; &#43; e, e);
  }
}
// 解析 properties 节点。
private void propertiesElement(XNode context) throws Exception {
  if (context != null) {
    //...省略
    parser.setVariables(defaults);
    configuration.setVariables(defaults);
  }
}
```

parse方法最终调用parseConfiguration解析不同的节点，比如解析properties节点，把解析出来的结果设置到 configuration配置文件中，到此配置解析结束。

### selectOne 一条SQL语句是如何执行

从获取SqlSession对象开始，前面已经完成了配置解析，下面先看下如何生成SqlSession对象。

先用解析好的Configuration对象生成SqlSessionFactory。

```java
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

再调用 openSession() 生成 SqlSession

```java
public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //通过事务工厂来产生一个事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //生成一个执行器(事务包含在执行器里)
      final Executor executor = configuration.newExecutor(tx, execType);
      //然后产生一个DefaultSqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      //如果打开事务出错，则关闭它
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException(&#34;Error opening session.  Cause: &#34; &#43; e, e);
    } finally {
      //最后清空错误上下文
      ErrorContext.instance().reset();
    }
  }
```

使用 DefaultSqlSession 实例化SqlSession对象，这里需要传入执一个执行器对象Executor。

执行器 Executor 实例化，直接Configuration类中的newExecutor方法产生。

```java
  //产生执行器
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    //这句再做一下保护,囧,防止粗心大意的人将defaultExecutorType设成null?
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //然后就是简单的3个分支，产生3种执行器BatchExecutor/ReuseExecutor/SimpleExecutor
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //如果要求缓存，生成另一种CachingExecutor(默认就是有缓存),装饰者模式,所以默认都是返回CachingExecutor
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    //此处调用插件,通过插件可以改变Executor行为
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

Configuration 中配置默认执行器类型是 ExecutorType.SIMPLE，对应的执行器是SimpleExecutor，由于一级缓存cacheEnabled开关状态默认为true, 最终生成的执行器是 CachingExecutor。

到此，SqlSession对象已经有，下面看一条SQL语句的执行过程。

执行SQL语句

```java
Blog blog = session.selectOne(&#34;com.example.mapper.BlogMapper.selectBlog&#34;, 1);
System.out.println(blog);
```

```java
public class DefaultSqlSession implements SqlSession {
  
    // 第一步
    @Override
    public &lt;T&gt; T selectOne(String statement, Object parameter) {
      List&lt;T&gt; list = this.&lt;T&gt;selectList(statement, parameter);
      if (list.size() == 1) {
        return list.get(0);
      } else if (list.size() &gt; 1) {
        throw new TooManyResultsException(&#34;Expected one result (or null) to be returned by selectOne(), but found: &#34; &#43; list.size());
      } else {
        return null;
      }
    }
  
  		// 第二步
      @Override
      public &lt;E&gt; List&lt;E&gt; selectList(String statement, Object parameter) {
        return this.selectList(statement, parameter, RowBounds.DEFAULT);
      }
  
      // 第三步
  		@Override
      public &lt;E&gt; List&lt;E&gt; selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
          //根据statement id找到对应的MappedStatement
          MappedStatement ms = configuration.getMappedStatement(statement);
          //转而用执行器来查询结果,注意这里传入的ResultHandler是null
          return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
          throw ExceptionFactory.wrapException(&#34;Error querying database.  Cause: &#34; &#43; e, e);
        } finally {
          ErrorContext.instance().reset();
        }
      }
}
```

从上面代码看到，selectOne查查最终委托各Executor对象的query方法来完成。

继续，Executor#query

前面在实例化Executor是我们已经知道最后生成的执行器是CachingExecutor。

```java
public class CachingExecutor implements Executor {
  // 第一步 
  @Override
  public &lt;E&gt; List&lt;E&gt; query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler 						resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    //query时传入一个cachekey参数
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  
  
  // 第二部
    @Override
  public &lt;E&gt; List&lt;E&gt; query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler      resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    //默认情况下是没有开启缓存的(二级缓存).要开启二级缓存,你需要在你的 SQL 映射文件中添加一行: &lt;cache/&gt;
    //简单的说，就是先查CacheKey，查不到再委托给实际的执行器去查
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() &amp;&amp; resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings(&#34;unchecked&#34;)
        List&lt;E&gt; list = (List&lt;E&gt;) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.&lt;E&gt; query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.&lt;E&gt; query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
}
```

先从二级Cache 中取，没有继续委托执行

```java
// 第一步
public abstract class BaseExecutor implements Executor {
  @Override
  public &lt;E&gt; List&lt;E&gt; query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity(&#34;executing a query&#34;).object(ms.getId());
    //如果已经关闭，报错
    if (closed) {
      throw new ExecutorException(&#34;Executor was closed.&#34;);
    }
    //先清局部缓存，再查询.但仅查询堆栈为0，才清。为了处理递归调用
    if (queryStack == 0 &amp;&amp; ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List&lt;E&gt; list;
    try {
      //加一,这样递归调用到上面的时候就不会再清局部缓存了
      queryStack&#43;&#43;;
      //先根据cachekey从localCache去查
      list = resultHandler == null ? (List&lt;E&gt;) localCache.getObject(key) : null;
      if (list != null) {
        //若查到localCache缓存，处理localOutputParameterCache
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //从数据库查
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      //清空堆栈
      queryStack--;
    }
    if (queryStack == 0) {
      //延迟加载队列中所有元素
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      //清空延迟加载队列
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
    	//如果是STATEMENT，清本地缓存
        clearLocalCache();
      }
    }
    return list;
  }
  
 // 第二步
 private &lt;E&gt; List&lt;E&gt; queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, 	 ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List&lt;E&gt;  list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    return list;
  }
 
}
```

```java
public class SimpleExecutor extends BaseExecutor {  
  // 第三步
  @Override
  public &lt;E&gt; List&lt;E&gt; doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      // 这个Configuration 和 BaseExecutor的Configuration是同一个对象
      Configuration configuration = ms.getConfiguration();
      //新建一个StatementHandler
      //这里看到ResultHandler传入了
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //准备语句
      stmt = prepareStatement(handler, ms.getStatementLog());
      //StatementHandler.query
      return handler.&lt;E&gt;query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
  
  // 连接数据库
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获得连接
    Connection connection = getConnection(statementLog);
    // 调用StatementHandler.prepare
    stmt = handler.prepare(connection);
    // 调用StatementHandler.parameterize
    handler.parameterize(stmt);
    return stmt;
  }
}
```

先执行 BaseExecutor  的 query 从缓存查询，缓存中查不到执行 queryFromDatabase方法，再调用 SimpleExecutor 的 doQuery。

在这里实例化了 StatementHandler 对象，获取数据库连接，最后由 StatementHandler 执行具体的SQL语句。

```java
public class SimpleStatementHandler extends BaseStatementHandler { 
@Override
  public &lt;E&gt; List&lt;E&gt; query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    // 执行SQL语句
    statement.execute(sql);
    //先执行Statement.execute，然后交给ResultSetHandler.handleResultSets
    return resultSetHandler.&lt;E&gt;handleResultSets(statement);
  }
}
```

DefaultResultSetHandler 处理执行结果并返回。

```java
public class DefaultResultSetHandler implements ResultSetHandler {
  @Override
  public List&lt;Object&gt; handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity(&#34;handling results&#34;).object(mappedStatement.getId());

    final List&lt;Object&gt; multipleResults = new ArrayList&lt;Object&gt;();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List&lt;ResultMap&gt; resultMaps = mappedStatement.getResultMaps();
    //一般resultMaps里只有一个元素
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null &amp;&amp; resultMapCount &gt; resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount&#43;&#43;;
    }

    String[] resultSets = mappedStatement.getResulSets();
    if (resultSets != null) {
      while (rsw != null &amp;&amp; resultSetCount &lt; resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount&#43;&#43;;
      }
    }

    return collapseSingleResultList(multipleResults);
  }

}
```

## MyBatis 源码完整流程图

![](https://raw.githubusercontent.com/telzhou618/images/main/img/MyBatis%E6%BA%90%E7%A0%81%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%20(1).png)

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/mybatis/  

