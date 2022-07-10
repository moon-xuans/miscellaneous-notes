# 3.二级缓存的组件
## 3.1.如何开启二级缓存
开启二级缓存的方式也比较简单，如下:
```xml
第一步: MyBatis 配置文件中配置
<settings>
      <setting name = "cacheEnabled" value = "true" />
</settings>

第二步: 在Mapper.xml文件中配置<cache/>标签, 一个Mapper.xml文件拥有唯一的namespace(命名空间)
<cache type="org.apache.ibatis.cache.impl.PerpetualCache" 
size="1024"  eviction="LRU"  flushInterval="120000"  readOnly="false" />
也可以配置<cache-ref/>,<cache-ref/>标签是为了引用其他的命名空间，那么当前命名空间将与引用的命名空间使用同一个缓存（对于同一命名空间下的多表查询可借助该标签避免脏读问题）
```
### 3.1.1.标签属性含义
在开启二级缓存的第二步中，要在Mapper.xml文件中配置标签，同时也可以为标签拥有的属性赋值，那标签的属性们的含义都是什么？
```
<settings>
      <setting name = "cacheEnabled" value = "true" />
</settings>

第二步: 在Mapper.xml文件中配置<cache/>标签, 一个Mapper.xml文件拥有唯一的namespace(命名空间)
<cache type="org.apache.ibatis.cache.impl.PerpetualCache" 
size="1024"  eviction="LRU"  flushInterval="120000"  readOnly="false" />
也可以配置<cache-ref/>,<cache-ref/>标签是为了引用其他的命名空间，那么当前命名空间将与引用的命名空间使用同一个缓存（对于同一命名空间下的多表查询可借助该标签避免脏读问题）
```
### 3.1.2.产生的效果
做了如上配置后产生的效果如下
```
a.映射语句文件中的所有 select 操作的结果将会被缓存。
b.映射语句文件中的所有 update操作( insert 、update 和 delete )会刷新缓存。
c.缓存会使用最近最少使用（LRU, Least Recently Used）算法来淘汰不需要的缓存。
d.缓存会间隔120000ms后清空一次缓存。
e.缓存会保存列表或对象（无论查询方法返回哪种）的 1024 个引用。
f.缓存会被视为读写缓存, 需要查询出来要被缓存的实体类实现Serializable接口。
  这意味着获取到的对象并不是共享的，可以安全地被调用者修改, 而不干扰其他调用者或线程 。
```
关于readOnly="false"为何需要查询出来的缓存实体类实现序列化接口：

这是因为二级缓存为了保证读写安全，开启了序列化功能，缓存中保存的不再是查询出的对象本身，而是查询出的对象进行序列化后的字节序列，在获取数据的时候，又会把存好的字节序列进行反序列化，克隆出新对象，进行返回。

所以对从二级缓存中得到数据做任何写操作，都不会影响到缓存中原有的对象，也就不会影响到其他来获取数据的调用者或线程。

> Java序列化就是指把Java对象转换为字节序列的过程。Java反序列化就是指把字节序列恢复为Java对象的过程。而在反序列化的时候会根据字节序列中保存的对象状态及描述信息，重建对象。

## 3.2.二级缓存组件结构
从以上的描述中我们可以看出，Mybatis的二级缓存要实现的功能更加复杂，比如：线程安全，过期清理，命中率统计，序列化....

Mybatis为了尽可能的职责分明的实现这些复杂逻辑，在这里使用了一种设计模式：**装饰者+责任链（变种）**，对二级缓存的功能组件进行设计。至于为什么说是一个责任链变种，我们需要先了解以下经典责任链的定义。
>责任链（经典定义）是一个请求有多个对象来处理，这些对象是一条链，但具体是由哪个对象来处理，根据条件来判断，如果不能处理会传递给该链的下一个对象，直到有对象处理它为止。

而责任链中的链，是如何形成的呢？举一个例子，比如我们的链式结构是a对象->b对象->c对象，那我们就让a对象持有b对象，b对象持有c对象。从a对象开始，a对象的方法中可以调用b对象的方法，而b对象的方法中也可以调用c对象的方法，通过这样的方式，便形成了一条责任链。

经典责任链的方式，要根据条件判断，虽然也许会经过链条上的很多对象，但最终只有一个对象真正对请求进行了处理，其他对象仅仅完成了向下传递，也要完成自己的功能实现。所以说Mybatis使用的是责任链的变种形式。

二级缓存的组件结构如下图所示：
![image-20220108102313004](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220108102313004.png)
二级缓存组件的顶级接口是Cache，定义了二级缓存的api，比如设置缓存，取出缓存。Cache下方有很多实现类，正是这些实现类形成责任链，组成了二级缓存。

实际上的结构是否如此呢，在获取二级缓存的时候，对二级缓存进行Debug，就可以印证我们刚才的说法了。
<img src="https://img-blog.csdnimg.cn/8ab53098bcf14296b9655a777ac8e937.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBATW9vbl94dWFu,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:200%;" />

可以看出，最上层是SynchronizedCache，持有了一个名为delegate的LoggingCache类型对象，以此类推，直到链条上的最后一个Cache的实现类--PerpetualCache。而PerpetualCache本身持有了一个HashMap，这才是二级缓存数据的真正存放地（缓存区）。

以查询为例，在调用二级缓存的getObject()方法的时候，就会从链条的起始端，比如SynchronizedCahce，开始调用自己特有的职能，另一个是调用链条上的下一个Cache实现类的getObject()方法，直到链条的尾端，比如PerpetualCache。调用链虽然复杂，但是每个实现类都是完成自己特有的附加功能，而最终真正完成数据存储工作的只有PerpetualCache这个类。

先来看下PerpetualCache这个类的源码，在这个类中的getObject()方法，仅仅是从map中取出数据。
```java
public class PerpetualCache implements Cache {

  private Map<Object, Object> cache = new HashMap<Object, Object>();

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }
```
而链条上的其他的Cache实现类是不是按照之前介绍的那样，做自己的功能并调用自己持有的链条上的下一个实现类的方法呢，我们也可以以几个实现类的源码为例来论证。比如：SynchronizedCache（负责线程安全）和LoggingCache(负责命中率统计）。

在查看SynchronizedCache类的源码的时候，不要忽略getObject方法上的synchronized关键字，这个方法在负责线程安全的问题后，便调用了责任链的下一个对象的getObject()方法。
```java
public class SynchronizedCache implements Cache {

  private final Cache delegate;

  @Override
  public synchronized Object getObject(Object key) {
  // 注意：这里！委派给下一个缓存实现类执行getObject()方法
    return delegate.getObject(key);
  }
```
LoggingCache的getObject()方法中，除了调用链条上的下一个对象的方法外，还会统计请求的次数和命中的次数，以此计算打印命中率。
```java
public class LoggingCache implements Cache {

  private final Cache delegate;
  
  @Override
  public Object getObject(Object key) {
    requests++; // 请求次数
    // 注意：看这里！委派给下一个缓存实现类执行getObject()方法
    final Object value = delegate.getObject(key);
    if (value != null) {
      hits++;
    }
    if (log.isDebugEnabled()) {
      log.debug("Cache Hit Ratio [" + getId() + "]: " + getHitRatio());
    }
    return value;
  }
```
## 3.3.事务缓存管理器
### 3.3.1.结构
我们都知道一个会话中的事务在未提交之前，其他会话是不允许读到它未提交的数据的。在未加入二级缓存之前，会话之间的都是如下图所示的样子，各自为政，互不干扰。
![在这里插入图片描述](https://img-blog.csdnimg.cn/fa11d4c75d2c4e8ba74b9e9e35ad3a98.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBATW9vbl94dWFu,size_20,color_FFFFFF,t_70,g_se,x_16)
二级缓存是可以跨会话的。那我们如果加入了二级缓存，并且按照缓存的一贯思路（进行查询操作的时候先查缓存，如果缓存中没有命中即查询数据库，并且把查到的结果缓存到二级缓存中）来做，会不会破坏原本隔离性，产生脏读？来看下面一张图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/3ceabcda4f0f49cc8626f49172a896e5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBATW9vbl94dWFu,size_20,color_FFFFFF,t_70,g_se,x_16)
会话1首先进行了修改操作，然后进行了查询操作，并且把查询后就把查到的结果放入缓存中，而此时会话2也进行了查询操作，就会查到缓存中的结果直接返回，尴尬的是会话1最终没有提交事务，选择了回滚。这样就造成了会话2读到的数据不准确，读到了会话1未提交的数据，产生了脏读。

所以Mybatis的二级缓存在设计时针对这样的情况，引入了**事务缓存管理器**。在事务缓存管理器中，维护了一个**本地暂存区**（会话范围内可见），本地暂存区又指向真正的**缓存区**（跨会话）。在进行查询操作的时候，会到缓存区中查看是否命中。如果没有命中，查询数据库得到数据后，仅仅把查询的结果放入暂存区，在提交事务的时候才要把暂存区中的数据刷新到缓存区。如果发生了回滚，则清空本地暂存区缓存的数据，不会刷新到缓存区，这样一来就避免了脏读的产生。

接下来我们先来通过部分源码了解一下事务管理器的结构：

从以下代码可以看出每个**CachingExecutor**对应一个事务缓存管理器，通过前面的学习我们知道，每个会话中持有一个**CachingExecutor**（缓存执行器）。所以每个**会话**都有自己单独的**事务缓存管理器**。
```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();
```
从以下代码我们得知，在事务缓存管理器中维护了一个HashMap，这个HashMap便是**暂存区的集合**，而且这个map的key是cache（缓存区），所以每一个缓存区都有相应的暂存区（TransactionalCache），放在map中作为键值对被**事务缓存管理器**所维护，因为每个会话都有自己单独的事务缓存管理器，作为管理器属性集合中的一个对象---暂存区也只是会话可见的。
```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();
```

接下来看一下代表暂存区的**TransactionalCache**，可以看见其中也维护了一个Map，这个map是暂存区真正用来**暂存数据**的地方，而**delegate**属性，代表的便是真正**缓存区**（刚刚介绍过的，Cache的实现类组成的责任链，完成了缓存区的维护），有了与缓存区之间的关联，在提交事务的时候，就可以方便的把暂存区的数据刷新到缓存区了。
```java
public class TransactionalCache implements Cache {
  private final Cache delegate;  // 指向缓存区
  private boolean clearOnCommit;
  private final Map<Object, Object>  entriesToAddOnCommit; // 暂存区
```
介绍完事务管理器，暂存区，缓存区之间的结构关系，我们来通过源码看下二级缓存进行查询和更新的过程。

### 3.3.2.查询
如果使用到二级缓存，在查询时，会调用二级缓存的query方法。这里主要看其中的**tcm.getObject(cache,key)**和**tcm.putObject(cache,key,list)**方法，一个是通过事务缓存管理器取数据的方法，一个是通过事务管理器放入数据的方法。
```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache(); // 获取缓存区
    if (cache != null) {
      flushCacheIfRequired(ms); // 刷新缓存存在
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
        // 注意：这里！（1）通过事务缓存管理器获取数据
        List<E> list = (List<E>) tcm.getObject(cache, key);
        // 如果二级缓存中没有查询到数据，则查询数据库
        if (list == null) {
        // 委托给BaseExecutor执行
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          // 注意：这里！（2）经过事务管理器放入数据
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
#### （1）tcm.getObject(cache,key)--->取出数据
在CachingExecutor的query()方法中，先是调用了事务缓存管理器的getObject(cache,key)方法。可以看见**TransactionalCacheManager**在处理getObject()的时候先调用了getTransactionalCache(),从map集合中取出当前缓存区对应的**TransactionalCache**(暂存区)，暂存区如果不存在，则创建一个新的暂存区对象存入map，然后调用获得的TransactionCache的getObject()方法。
```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();


  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }

  private TransactionalCache getTransactionalCache(Cache cache) {
    TransactionalCache txCache = transactionalCaches.get(cache);
    if (txCache == null) {
      txCache = new TransactionalCache(cache);
      transactionalCaches.put(cache, txCache);
    }
    return txCache;
  }
```
在TransactionalCache的getObject()方法中，直接调用了其指向的**缓存区**的getObject()方法，**说明二级缓存在获取数据的时候会直接去缓存区（跨会话）取数据**。

而在clearOnCommit这个布尔值为true的时候，即使缓存区命中数据也只能返回null，这是因为，只有在**有更新操作且未提交**的时候clearOnCommit才是true，这种状态对于当前会话当前事务来说，缓存区的数据已经不准确了，所以最好的选择是重新查询数据库。
```java
public class TransactionalCache implements Cache {

  private final Cache delegate; // 指向缓存区（链条式的Cache实现类）
  private boolean clearOnCommit; // 执行更新后clearOnCommit将变为true
  private final Map<Object, Object> entriesToAddOnCommit; // 本地暂存
  
  // 获取缓存数据，从缓存区去查询
  @Override
  public Object getObject(Object key) {
    // issue #116
    Object object = delegate.getObject(key);
    if (object == null) { 
      entriesMissedInCache.add(key);
    }
    // issue #146
    if (clearOnCommit) { // 如果更新了数据，缓存区就算有数据也要返回空，要去数据库中去取数据
      return null;
    } else {
      return object;
    }
  }
```
#### （2）tcm.putObject(cache,key,list)--->放入数据

在query()方法中，没有从缓存区中取到数据，而重新查询了数据的情况下，就要调用tcm.putObject(),通过事务管理器设置数据到缓存。与getObject()一样，TransactionalCacheManager的putObject()方法也要先调用getTransactionalCache()获得**TransactionalCache**(暂存区)，然后调用TransactionalCache的putObject()方法。
```java
public class TransactionalCacheManager {

  @Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object); // 存数据，存到暂存区
  }
```
### 3.3.3.提交
在提交的方法中，我们会**把暂存区中的所有内容刷新到缓存区中**。

在我们调用sqlSession.commit()方法的时候，也会调用当前会话持有的缓存执行器的commit()方法，缓存执行器会执行事务缓存管理器的commit()方法。看一下事务缓存管理器的提交的源码，在事务缓存管理器的commit()方法，会调用事务缓存管理器所有暂存区(TransactionalCache)的commit()方法。
```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();

  public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }
```
在TransactionalCache的commit()方法中，如果有未提交的更新操作（clearOnCommit为true），则要清空缓存区，因为更新后，缓存区的数据便是不准确的了。随后调用flushPendingEntries()和reset()两个方法，flushPendingEntries()方法负责把所有暂存区的内容刷新到缓存中。而reset()方法则负责把本地暂存区清空，同时把clearOnCommit置为false。
```java
public class TransactionalCache implements Cache {

  private final Cache delegate; // 指向缓存区（链条式的Cache实现类）
  private boolean clearOnCommit; // 执行更新后clearOnCommit将变为true
  private final Map<Object, Object> entriesToAddOnCommit; // 本地暂存
 
 public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
  }

  private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
  }

  private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }
```
### 3.3.4.更新
在缓存执行器调用更新操作的时候，会调用flushCacheIfRequired(),这个方法中会先判断ms.isFlushCacheRequired()，为true并且二级缓存存在就会执行事务缓存执行器的clear()方法，而isFlushCachingRequired()就是从标签里面取到的flushCache的值。而增删改操作的flushCache属性默认为true。所以进行更新的时候，也会调用事务缓存管理器的clear方法。
```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  @Override
  public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    flushCacheIfRequired(ms);
    return delegate.update(ms, parameterObject);
  }
  
 private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    if (cache != null && ms.isFlushCacheRequired()) {      
      tcm.clear(cache);
    }
  }
```
在TransactionalCacheManager的clear方法中。依然是先获取暂存区，并调用暂存区的clear()方法。
```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();

  public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
  }
  
  private TransactionalCache getTransactionalCache(Cache cache) {
    TransactionalCache txCache = transactionalCaches.get(cache);
    if (txCache == null) {
      txCache = new TransactionalCache(cache);
      transactionalCaches.put(cache, txCache);
    }
    return txCache;
  }
```
TransactionalCache的clear()方法中，clearOnCommit属性被置为了true，并清空了暂存区。清空暂存区不难理解，因为如果存在更新操作，则暂存区暂存起来的数据则有可能不再准确了。并且缓存区也定然出现了不一致的情况，所以在TransactionalCache的commit方法中，会去判断clearOnCommit是否为true(即是否进行过更新操作)，如果是，缓存区的数据也会被clear()掉。而在清除执行完成后，reset()方法中会把clearOnCommit重新置为false。

### 3.3.4.总结
Mybatis使用了装饰者+责任链（变种）的模式构建了二级缓存的组件，每一个功能都有相应的Cache实现类来完成，同时这些实现类也会调用自己持有的Cache实现类，完成责任链。最终被调用的类是PerpetualCache，它就是最终负责数据存储的类。

而为了解决二级缓存跨会话使用可能引起的脏读问题，mybatis引入了事务缓存管理器，每一个会话持有一个事务缓存管理器，每个事务缓存管理器维护着多个缓存区（每个namespace都有对应的缓存区）对应的暂存区，暂存区中维护本地暂存数据，并指向它所属的缓存区。

通过事务缓存管理器查询的时候，直接去查缓存区，但是如果没有命中，重新查询出的数据仅放入暂存区，直到进提交，才把数据刷新到缓存区。这是为了防止其他会话查到当前会话中的事务未提交的数据。而在执行更新操作的时候，会先清空对应的暂存区数据，在提交事务的时候，也会把对应的缓存区数据清空。