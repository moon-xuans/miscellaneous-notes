# 如何保证SqlSession的线程安全

## 1.前言

这几天，在做自实现mybaits的改进的过程中，突然想到之前有个面试官问的sqlSession的安全问题。因此，在网上找了一些相关问题，总结在这里了。

## 2.DefaultSqlSession是线程不安全的

在Mybatis中SqlSession是提供给外部调用的顶层接口，实现类有:DefaultSqlSessoin、SqlSessionManager以及mybatis-spring提供的实现SqlSessionTemplate。默认实现类为DefaultSqlSession，是线程不安全的。

![image-20220405103529550](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405103529550.png)

对于Mybatis提供的原生实现类来说，用的最多就是DefaultSqlSession，但是查看源码之后，知道DefaultSqlSession这个类不是线程安全的！

![image-20220405103723539](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405103723539.png)

因此，那我们在使用过程中，是如何保证DefaultSqlSession线程安全的？看下面的文章。

## 3.SqlSessionTemplate是如何保证线程安全的

在我们平时的开发中通常会用到Spring，也会用到mybatis-spring框架，在Spring集成Mybatis的时候我们可以用到SqlSessionTemplate(Spring提供的SqlSession实现类)。

我们看下SqlSessionTemplate的源码注释:

![image-20220405104413874](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405104413874.png)

通过源码注释可以看到SqlSessionTemplate是线程安全的类，并且实现了SqlSession接口，也就是说我们可以通过SqlSessionTemplate来代替以往的DefaultSqlSession完成对数据库CRUD操作，并且还保证单例线程安全，那么它是如何保证线程安全的呢？

首先，通过SqlSessionTemplate拥有的三个重载的构造方法分析，最终都会调用最后一个构造方法，会初始化一个SqlSessionProxy的代理对象，如果调用代理类实例中发现的SqlSession接口中定义的方法，该调用会被导向SqlSessionInterceptor的invoke方法触发代理逻辑。

![image-20220405105009665](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405105009665.png)

接下来看看SqlSessionInterceptor的invoke方法

1.通过getSqlSession方法获取SqlSession对象(如果使用了事务，从Spring事务上下文获取)

2.调用SqlSession的接口方法操作数据库获取结果

3.返回结果集

4.若发生异常则转换后抛出异常，并最终关闭了SqlSession对象

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 获取SqlSession(这个sqlSession才是真正使用的，它不是线程安全的)
    // 这个方法可以根据Spring的事务上下文来获取事务范围内的SqlSession
    SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

    Object unwrapped;
    try {
        // 调用SqlSession对象的方法(select、update等)
        Object result = method.invoke(sqlSession, args);
        // 判断是否为事务操作，如果未被spring事务托管则自动提交commit
        if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
            sqlSession.commit(true);
        }

        unwrapped = result;
    } catch (Throwable var11) {
        // 如果出现异常则根据情况转换后抛出
        unwrapped = ExceptionUtil.unwrapThrowable(var11);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
            SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            sqlSession = null;
            Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException)unwrapped);
            if (translated != null) {
                unwrapped = translated;
            }
        }

        throw (Throwable)unwrapped;
    } finally {
        // 最终关闭sqlSession对象
        if (sqlSession != null) {
            SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }

    }

    return unwrapped;
}
```

再重点分析下getSqlSession方法:

1.若无法从当前线程的ThreadLocal中获取，则通过SqlSessionFactory获取SqlSession

2.若开启了事务，则从当前线程的ThreadLocal上下文中获取SqlSessionHolder

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    Assert.notNull(sessionFactory, "No SqlSessionFactory specified");
    Assert.notNull(executorType, "No ExecutorType specified");
    // 若开启了事务支持，则从当前的ThreadLocal上下文中获取SqlSessionHolder
    // SqlSessionHolder是SqlSession的包装类
    SqlSessionHolder holder = (SqlSessionHolder)TransactionSynchronizationManager.getResource(sessionFactory);
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
        return session;
    } else {
        LOGGER.debug(() -> {
            return "Creating a new SqlSession";
        });
        // 若无法从ThreadLocal上下文中获取则通过SqlSessionFactory获取SqlSession
        session = sessionFactory.openSession(executorType);
        // 若为事务操作，则注册SqlSessionHolader到ThreadLocal中
        registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
        return session;
    }
}
```

SqlSessionTemplate的流程大概就是这样，通过使用同一个事务的connection来保证线程安全。

## 4.SqlSessionManager又是什么？

SqlSessionManager是Mybatis提供的线程安全的操作类

![image-20220405141737985](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405141737985.png)

通过上图可以发现SqlSessionManager的构造方法是private的，那应该怎么创建呢？其实SqlSessionManager创建对象是通过newInstance方法创建对象的，但需要注入它虽然是私有的构造方法，并且提供我们一个共有的instance方法，但它并不是一个单例模式。

SqlSessionManager的openSession方法及其重载方法是直接通过调用底层封装SqlSessionFactory对象的openSession方法来创建SqlSession对象的，如下所示:

![image-20220405144146750](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405144146750.png)

SqlSessionManager中实现SqlSession接口中的方法，例如:select、update等，都是直接调用SqlSessionProxy代理对象中相应的方法，在创建该代理对象的时候使用的InvocationHandler对象是SqlSessionInterceptor,它是定义在SqlSessionManager的一个内部类，其定义如下:

```java
private class SqlSessionInterceptor implements InvocationHandler {
  public SqlSessionInterceptor() {
      // Prevent Synthetic Access
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 获取当前ThreadLocal上下文的SqlSession
    final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
    if (sqlSession != null) {
      try {
          // 从上下文获取到SqlSession之后调用对应的方法
        return method.invoke(sqlSession, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    } else {
        // 如果无法从ThreadLocal上下文中获取SqlSession则新建一个SqlSession
      try (SqlSession autoSqlSession = openSession()) {
        try {
          final Object result = method.invoke(autoSqlSession, args);
          autoSqlSession.commit();
          return result;
        } catch (Throwable t) {
          autoSqlSession.rollback();
          throw ExceptionUtil.unwrapThrowable(t);
        }
      }
    }
  }
}
```

此处在思考下ThreadLocal的localSqlSession对象在什么时候赋值对应的SqlSession，往上查找最终定位代码(若调用startManagerSession方法将设置ThreadLocal的localSqlSession上下文中的SqlSession对象)，如下

![image-20220405144819478](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220405144819478.png)

## 5.SqlSessionTemplate与SqlSessionManager的联系与区别

- SqlSessionTemplate是Mybatis为了接入Spring提供的Bean。通过TransactionSynchronizationManager中的ThreadLocal<Map<Object,Object>>保存线程对应的SqlSession，实现session的线程安全。
- SqlSessionManager是Mybatis不接入Spring时用于管理SqlSession的Bean。通过SqlSessionManager的ThreadLocal实现session的线程安全。

## 6.总结

通过上面的代码分析，我们可以看出Spring解决SqlSession线程安全问题的思路就是动态代理与ThreadLocal的运用，我们可以触类旁通:当遇到线程不安全的类，但是又想当做线程安全的类使用，则可以使用ThreadLocal进行线程上下文的隔离，此处的动态代理技术更好的解决了上层API调用的非侵入性，保证API接口调用的**高内聚、低耦合原则**。