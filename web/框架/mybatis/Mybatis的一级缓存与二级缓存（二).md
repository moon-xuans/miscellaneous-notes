# 2.一级缓存与二级缓存的结构关系
## 2.1.思维发散
二级缓存是用来解决一级缓存不能跨会话共享的问题，范围是namespace级别,可以被多个sqlSession(会话)共享，生命周期和应用同步。默认关闭。

通过对一级缓存的学习我们知道，一级缓存是默认开启的，如果我们同时开启了二级缓存，那就势必存在一级缓存he二级缓存都要使用的情况。这样一来我们要思考的第一个问题产生了，一级缓存和二级缓存的执行顺序是怎样的呢？

还是先推断一下，二级缓存作为一个作用范围更广的缓存（可以跨会话），从节省资源的角度来设计，二级缓存肯定要工作在一级缓存（不能跨会话）之前的。也就是只有取不到二级缓存的情况下才到一个会话中取一级缓存。如果你的Mybatis使用了二级缓存，那么在执行select查询的时候，Mybatis会先从二级缓存中取数据，取不到才会走一级缓存，一级缓存也取不到，就会与数据库进行交互。即Mybatis查询数据的顺序是:`二级缓存 -> 一级缓存 -> 数据库`。

按照这种执行顺序设计来思考，一级缓存已经利用BaseExecutor完成了自身功能的实现，那么二级缓存要加在哪里进行维护，才合适呢？实际上Mybatis这里用了一个设计模式--`装饰器模式`来维护二级缓存，实现这个功能的类就是`CachingExecutor`(缓存执行器)。

> 装饰器模式（Decorator Pattern）允许向一个现有的对象添加新的功能，同时又不改变其结构。这种类型的设计模式属于结构型模式，它是作为现有的类的一个包装

具体是怎么做的呢？Mybatis让`CachingExecutor`对`BaseExecutor`进行了包装。`CachingExecutor`中不仅要实现了二级缓存的功能，同时也要持有一个基础执行器`(BaseExecutor)`。当查询请求来临的时候，`CachingExecutor`会先判断二级缓存是否有缓存结果，如果有就直接返回，如果没有则委派给自己持有的BaseExecutor实现类，比如`SimpleExecutor`(简单执行器)来执行查询，这样就顺理成章的从二级缓存过渡到了一级缓存的执行流程了。最后会把得到的结果缓存起来，并且返回给用户。

## 2.2.源码论证
（1）还是要从创建会话讲起

创建会话的过程中，会先创建执行器,这个执行器便是`CachingExecutor`了。然后把得到的执行器--`CachingExecutor`交给`DefaultSqlSession`(会话)来持有。

```java

public class DefaultSqlSessionFactory implements SqlSessionFactory {

  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
    try {
      boolean autoCommit;
      try {
        autoCommit = connection.getAutoCommit();
      } catch (SQLException e) {
        // Failover to true, as most poor drivers
        // or databases won't support transactions
        autoCommit = true;
      }      
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      final Transaction tx = transactionFactory.newTransaction(connection);
      // 注意：这里！创建Executor执行器，这里创建的是CachingExecutor
      final Executor executor = configuration.newExecutor(tx, execType);
      // 创建DefaultSqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
（2）创建执行器部分源码。
在`newExecutor（）`方法中，首先还是要创建基础执行器(`BaseExecutor`)的子类，毕竟一级缓存的逻辑还要依靠它去完成。如果开启了缓存，最后要创建一个缓存执行器(`CachingExecutor`)，并把之前创建好的基础执行器(`BaseExecutor`)的子类作为`CachingExecutor`的有参构造的参数传入，让`CachingExecutor`持有`BaseExecutor`。并返回`CachingExecutor`让`DefaultSqlSession`（会话）来持有。

```java
public class Configuration {

 public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
（3）执行查询部分源码
Mybatis是以`sqlSession.selectList()`方法，作为查询的入口。可以看见，在`selectList()`方法中，调用了`executor.query()`方法来获取数据，而`sqlSession`持有的`Executor`正是缓存执行器(`cachingExecutor`)。也就是说这里调用的是`CachingExecutor`的`query()`方法。

```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;


  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      // 注意：这里！调用执行器的查询方法
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

```
在`CachingExecutor`的`query()`方法中，如果使用了二级缓存并且二级缓存存在，则先去二级缓存中查找数据，如果数据存在则返回数据。如果数据不存在，会直接委派给一级缓存进行查询。
```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager(); // 注意：这里！这里持有的便是基础执行器的子类

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
      // 获取二级缓存
    Cache cache = ms.getCache();
    if (cache != null) {
    // 刷新二级缓存
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
      	// 注意：这里！从二级缓存中查询数据
        List<E> list = (List<E>) tcm.getObject(cache, key);
        // 注意：这里！二级缓存中没有数据，委托给BaseExecutor执行
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    // 委托给BaseExecutor执行
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

```

## 2.3.结构与总结
总结：`SqlSession`持有`CachingExecutor`,`CachingExecutor`来完成二级缓存的功能实现，并且持有`BaseExecutor`，在二级缓存开启并且查不到数据时（或者二级缓存本身没有开启），都会委派给`BaseExecutor`来执行查询。
整体结构图如下：

![image-20220108102200129](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220108102200129.png)