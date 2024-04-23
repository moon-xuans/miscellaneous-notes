# 1.mybatis一级缓存源码分析
## 1.1.为什么要有一级缓存
每当我们使用Mybatis开启一次和数据库的会话，就会创建一个SqlSession对象来表示这个会话。就在这一次会话汇总，我们有可能反复执行完全相同的查询语句，这些相同的查询语句在没有执行过更新的情况下返回的结果是一致的。如果每次都去和数据库进行交互查询的话，就会造成资源浪费。所以，mybatis加入了一级缓存，用来在一次会话中缓存查询结果。

总结下一级缓存的存在起到的作用：在同一个会话里面，多次执行相同的sql语句（statementId,参数,rowBounds完全相同）,会直接从内存取到缓存的结果，不会再发送到数据库与数据库交互。但是不同的会话里面，即使执行的sql一模一样，也不能使用到一级缓存。

## 1.2.一级缓存与会话的关系
一级缓存也叫本地缓存，Mybatis的一级缓存是在==会话层面==(SqlSession)进行缓存的。==默认开启==，不需要任何的配置。

首先我们先思考一个问题，在Mybatis执行的流程里面，涉及到这么多对象，那么缓存Cache应该放在哪个对象里面去维护？

先来进行以下推断，我们已经知道一级缓存的作用范围是会话，那么这个对象肯定是在SqlSession里面创建的，作为SqlSession的一个属性存在。SqlSession本身是一个接口，它的实现类DefaultSqlSession里面只有两个属性---Configuration和Executor。Configuration是全局的，与我们知道的一级缓存的作用范围不符，所以缓存只可能放在Executor里面维护---而事实也正是如此，SimpleExecuotr/ReuseExecutor/BatchExecutor的父类==BaseExecutor的构建函数中就持有了Cache==。

直接看源码

（1）创建会话的源码部分：

首先是调用DefaultSqlSessionFactory的 `openSession()`方法，即：开启会话

`openSession（）` 方法中调用了 `openSessionFromDataSource()` 方法，openSessionFromDataSource()方法中先是调用  `configuration.newExecutor(tx,execType)` 创建了执行器(executor),然后调用 `DefaultSqlSession` 的构造器方法，并闯入创建好的执行器(executor)，这样就创建出了DefaultSqlSession对象并让其持有了executor属性。
```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {

// 创建会话的方法
@Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //注意：看这里！创建Executor执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      //注意：看这里！创建DefaultSqlSession,executor作为DefaultSqlSession构造方法的一个参数传入
      // DefaultSqlSession持有了Executor
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

```

（2）创建执行部分源码
而创建执行器的时候，会根据具体传入的执行器（executor）的类型，来选择一个合适的执行器（executor）创建出来。但是不管最终选择哪个执行器，他们都是`BaseExecutor`的子类(缓存执行器除外），而我们的一级缓存，正是BaseExecutor的一个属性，而创建好的执行器作为BaseExecutor的子类也有着父类的属性。所以 ==SqlSession对象持有了executor属性，而executor持有了一级缓存==。 我们之前的一级缓存与会话的关系也得到了印证。
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
    // 批处理执行器
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
    // 可重用执行器
      executor = new ReuseExecutor(this, transaction);
    } else {
    // 简单执行器
      executor = new SimpleExecutor(this, transaction);
    }
    // 如果开启缓存，则使用缓存执行器（是关于二级缓存的）
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
看下 `BaseExecutor` 的属性，它持有了`PerpetualCache`，也就是一级缓存。
```java
public abstract class BaseExecutor implements Executor {

  protected PerpetualCache localCache;
```
既然` PerpetualCache` 就是一级缓存了，那我们看看一级缓存到底是什么，最终这些东西都存在了一个`HashMap`里面。
```java
public class PerpetualCache implements Cache {

  private final String id;

	// 一级缓存最终存入容器
  private Map<Object, Object> cache = new HashMap<Object, Object>();

  public PerpetualCache(String id) {
    this.id = id;
  }

  @Override
  public String getId() {
    return id;
  }

  @Override
  public int getSize() {
    return cache.size();
  }

  @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }

  @Override
  public Object removeObject(Object key) {
    return cache.remove(key);
  }

  @Override
  public void clear() {
    cache.clear();
  }
}
```
## 1.3.一级缓存的生命周期
- 当会话结束时，SqlSession对象及其内部的Executor对象还有Cache对象也一并释放掉。
- 如果SqlSession调用了close（）方法，会释放掉一级缓存Cache对象，一级缓存将不可用；
- 如果SqlSession调用了clearCache()，会清空Cache对象中的数据，但是该对象仍可使用;
- SqlSession中执行了任何一个update操作(update()、delete()、insert())，都会清空Cache对象的数据，但是该对象可以继续使用。

## 1.4.一级缓存的执行流程概要
缓存执行的大概思路与我们熟知的缓存思想一致。
1.对于某个查询，根据statementId,params,rowBounds来构建一个key值，根据这个key值去缓存Cache取出对应的key值存储的缓存结果
2.判断Cache中根据特定的key值取的数据是否为空，即是否命中；
3.如果命中，则直接将缓存结果返回；
4.如果没命中：
去数据库中查询数据，得到查询结果；
	a.将key和查询的记过分别作为key，value对存储到Cache中
	b.将查询结果返回;

具体如何实现，直接看源码
（1）查询入口：
可以看见，查询最终是调用了`DefaultSqlSession`持有的属性`executor`的`query()`方法。
```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
    // 根据传入的statementId,获取MappedStatement对象
      MappedStatement ms = configuration.getMappedStatement(statement);
      // RowBounds是用来逻辑分页（按照条件将数据库查询到内存中，在内存中进行分页）
      // wrapCollection(parameter)是用来装饰集合或和数组参数
      // 注意：看这里！调用执行器的查询方法
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
executor的`query()`方法进行了一级缓存的逻辑，会调用`localCache.getObject(key)`从缓存中获取数据，如果获取不到，又会调用`queryFromDatabase()`方法。见名知意，这个方法就是用来与数据库进行交互取数据。

在`queryFromDatabase()`方法中，调用`doQuery()`来执行查询，再把得到的结果调用`localCache.putObject(key, list)`放入一级缓存。

如果我们继续查看`doQuery()`方法，就会发现这个方法是==抽象==的,这里涉及到一个常用的设计模式: ==模板模式== 。真正的`doQuery()`方法的实现是在`baseExecutor的子类方法`中去完成的，完成数据库中查询数据封装数据的部分。

> 模板模式(Template Pattern):一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方法进行。

```java
public abstract class BaseExecutor implements Executor {

 protected PerpetualCache localCache;

@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache(); // 清除缓存
    }
    List<E> list;
    try {
      queryStack++;
      // 注意：这里！从一级缓存中获取数据
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
      // 注意：这里！如果一级缓存没有数据，则从数据库查询数据
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
  
 private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
    // 注意：这里！执行查询
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    // 注意：这里！放入缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }

// 注意：这里！这是一个抽象的方法，等着子类去实现
  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;
```
## 1.5.结构与总结
小结：`sqlSession`持有`BaseExecutor`,`BaseExecutor`持有了`一级缓存`，查询时调用BaseExecutor的`query()`方法，并在==query()方法中完成了一级缓存的功能==。

缓存查到了就返回查询结果，查询不到就调用`queryFromDatabase()`方法，然后queryFromDatabase()方法中调用`doQuery()`方法从数据库中查询数据，然后放入一级缓存，其中`doQuery()方法是抽象的`,需要`BaseExecutor的不同类型子类`具体实现。

整体结构图如下：
![image-20220108102111005](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220108102111005.png)