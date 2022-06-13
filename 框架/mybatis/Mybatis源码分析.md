# 1.Mybatis源码分析

![image-20211203152327266]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211203152327266.png)

## 1.1.获取数据流

### 1.1.1.通过资源文件获取数据流

```java
String resource = "sqlMapConfig.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
```

Resources是一个辅助类，大多数操作都是通过classLoaderWrapper去做IO相关的事情

![image-20211203153028641]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211203153028641.png)



调用classLoaderWrapper.getResourceAsStream(resource,loader)方法

```java
public static InputStream getResourceAsStream(ClassLoader loader, String resource) throws IOException {
    InputStream in = classLoaderWrapper.getResourceAsStream(resource, loader);
    if (in == null) {
      throw new IOException("Could not find resource " + resource);
    }
    return in;
  }
```

```java
public InputStream getResourceAsStream(String resource, ClassLoader classLoader) {
  return getResourceAsStream(resource, getClassLoaders(classLoader));
}
```
在这里首先会获取它的所有类加载器

```java
//一共5个类加载器
  ClassLoader[] getClassLoaders(ClassLoader classLoader) {
    return new ClassLoader[]{
        classLoader,
        defaultClassLoader,
        Thread.currentThread().getContextClassLoader(),
        getClass().getClassLoader(),
        systemClassLoader};
  }
```

用5个类加载器一个个查找资源，只要其中一个找到，就返回

```java
InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
    for (ClassLoader cl : classLoader) {
      if (null != cl) {

        // try to find the resource as passed
        InputStream returnValue = cl.getResourceAsStream(resource);

        // now, some class loaders want this leading "/", so we'll add it and try again if we didn't find the resource
        // 现在，很多加载器都想要加上'/'，因此我们加上，在试一次
        if (null == returnValue) {
          returnValue = cl.getResourceAsStream("/" + resource);
        }

        if (null != returnValue) {
          return returnValue;
        }
      }
    }
    return null;
  }
```

## 1.2.获取SessionFactory

![image-20211203155653120]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211203155653120.png)

首先通过SqlSessionFactoryBuilder的默认构造方法创建一个实例对象，然后通过build方法创建SessionFactory

```java
 public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
  }
```

```java
// 上面的方法都会合流到这里
// 它使用了一个参照了XML文档或更特定的SqlMapConfig.xml文件的Reader实例.
// 可选的参数是environment和properties。Environment决定加载哪种环境（开发环境/生产环境），包括数据源和事务管理器。
// 如果使用properties,那么就会加载那些properties(属性配置文件)，那些属性可以用${propName}语法形式多次用到配置文件中。和spring很像。
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      // 委托XMLConfigBuilder来解析xml文件，并构建
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
```

调用XMLConfigBuilder的构造方法

```java
// 构造函数，转换成XPathParser再去调用构造函数
public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    // 构造一个需要验证，XMLMapperEntityResolver的XPathParser
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
  }
```

实例一个XMLMapperEntityResovler

```java
// 目的是未联网的情况下也能做DTD验证，实现原理就是将DTD搞到本地，然后用org.xml.sax.EntityResovler,最后调用DocumentBuilder.setEntityResovler来达到脱机验证
public class XMLMapperEntityResolver implements EntityResolver {
```

调用XPathParser的构造方法

```java
// XPath解析器，用的都是JDK的类包，封装了一下，使得用起来更方便
public class XPathParser {
```

传入是否需要验证参数，Properties，EntityResovler

```java
 public XPathParser(InputStream inputStream, boolean validation, Properties variables, EntityResolver entityResolver) {
    commonConstructor(validation, variables, entityResolver);
    this.document = createDocument(new InputSource(inputStream));
  }
```

调用一个初始化方法

```java
private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
    this.validation = validation;
    this.entityResolver = entityResolver;
    this.variables = variables;
	//共通构造函数，除了把参数都设置到实例变量里面去以外，还初始化了XPath
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
  }
```

调用createDocument方法

调用builder.parse()解析输入源

```java
private Document createDocument(InputSource inputSource) {
    // important: this must only be called AFTER common constructor
    try {
		//这个是DOM解析方式
      DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
      factory.setValidating(validation);

		//名称空间
      factory.setNamespaceAware(false);
		//忽略注释
      factory.setIgnoringComments(true);
		//忽略空白
      factory.setIgnoringElementContentWhitespace(false);
		//把 CDATA 节点转换为 Text 节点
      factory.setCoalescing(false);
		//扩展实体引用
      factory.setExpandEntityReferences(true);

      DocumentBuilder builder = factory.newDocumentBuilder();
		//需要注意的就是定义了EntityResolver(XMLMapperEntityResolver)，这样不用联网去获取DTD，
		//将DTD放在org\apache\ibatis\builder\xml\mybatis-3-config.dtd,来达到验证xml合法性的目的
      builder.setEntityResolver(entityResolver);
      builder.setErrorHandler(new ErrorHandler() {
        @Override
        public void error(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void fatalError(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void warning(SAXParseException exception) throws SAXException {
        }
      });
      return builder.parse(inputSource);
    } catch (Exception e) {
      throw new BuilderException("Error creating document instance.  Cause: " + e, e);
    }
  }
```

调用XMLConfigBuilder的构造方法

```java
// XML配置构建起，建造者模式，继承BaseBuilder
public class XMLConfigBuilder extends BaseBuilder {
   
// 构建器的基类，建造者模式
public abstract class BaseBuilder {
```



```java
// 上面的构造函数最后都合流到这个函数，传入XPathParser
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
  // 首先调用父类初始化Confiuration
  super(new Configuration());
  // 错误上下文设置成SQL Mapper Configuration(XML文件配置)，以便后界面出错了报错
  ErrorContext.instance().resource("SQL Mapper Configuration");
  // 将Properties全部设置到Configuration里面去
  this.configuration.setVariables(props);
  this.parsed = false;
  this.environment = environment;
  this.parser = parser;
}
```

初始化Configuration

```java
public Configuration() {
  //注册更多的类型别名，至于为何不直接在TypeAliasRegistry里注册，还需研究
  typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
  typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);

  //....
  // 注册了大量的类型别名
  typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
  typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
  typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
  typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
  typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);

  typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
  typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);

  languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
  languageRegistry.register(RawLanguageDriver.class);
}
```

调用父类的构造方法，需要创建Configuration的实例对象，并把Configuration的类型注册机和类型处理器注册赋给BaseBuilder

```java
public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
```

解析xml文件

```java
XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
 return build(parser.parse());
```

进行解析配置

```java
// 解析配置 
public Configuration parse() {
    // 如果已经解析过了，报错
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    //  <?xml version="1.0" encoding="UTF-8" ?> 
    //  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" 
    //  "http://mybatis.org/dtd/mybatis-3-config.dtd"> 
    //  <configuration> 
    //  <environments default="development"> 
    //  <environment id="development"> 
    //  <transactionManager type="JDBC"/> 
    //  <dataSource type="POOLED"> 
    //  <property name="driver" value="${driver}"/> 
    //  <property name="url" value="${url}"/> 
    //  <property name="username" value="${username}"/> 
    //  <property name="password" value="${password}"/> 
    //  </dataSource> 
    //  </environment> 
    //  </environments>
    //  <mappers> 
    //  <mapper resource="org/mybatis/example/BlogMapper.xml"/> 
    //  </mappers> 
    //  </configuration>
    
    // 根节点是configuration
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```

解析node节点

```java
 public XNode evalNode(String expression) {
    return evalNode(document, expression);
  }

	//返回节点
  public XNode evalNode(Object root, String expression) {
    Node node = (Node) evaluate(expression, root, XPathConstants.NODE);
    if (node == null) {
      return null;
    }
    return new XNode(this, node, variables);
  }

 private Object evaluate(String expression, Object root, QName returnType) {
    try {
        // 最终合流到这儿，直接调用XPath.evaluate
      return xpath.evaluate(expression, root, returnType);
    } catch (Exception e) {
      throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
    }
  }
```

构造XNode节点

```java
// 对org.w3c.dom.Node的包装
public class XNode {
    
    
  //在构造时就把一些信息（属性，body）全部解析好，以便我们直接通过getter函数取得
  public XNode(XPathParser xpathParser, Node node, Properties variables) {
    this.xpathParser = xpathParser;
    this.node = node;
    this.name = node.getNodeName();
    this.variables = variables;
    this.attributes = parseAttributes(node);
    this.body = parseBody(node);
  }
```

进行构建

```java
XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
return build(parser.parse());

// 最后一个build方法使用了一个Configuration作为参数，并返回DefaultSqlSessionFactory
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }

```

## 1.3.获取一个session

![image-20211203171034434]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211203171034434.png)

```java
// 构建SqlSession的工厂，工厂模式
public interface SqlSessionFactory {
    
// 默认的SqlSessionFactory
public class DefaultSqlSessionFactory implements SqlSessionFactory {
```

打开一个Session

```java
// 最终都会调用2种方法，openSessionFromDataSource,openSessionFromConnection
@Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
```

获取默认执行器

```java
// 默认为简单执行器
protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;

public ExecutorType getDefaultExecutorType() {
    return defaultExecutorType;
  }
```

通过数据源打开session

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        // 通过事务工厂产生一个事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        // 生成一个执行器（事务包含在执行器里）
      final Executor executor = configuration.newExecutor(tx, execType);
        // 然后生成一个DefaultSqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        // 如果打开事务出错，则关闭它
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        // 最后清空错误上下文
      ErrorContext.instance().reset();
    }
  }
```

获取事务工厂

```java
 private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
     // 如果没有配置事务工厂，则返回事务工厂
    if (environment == null || environment.getTransactionFactory() == null) {
      return new ManagedTransactionFactory();
    }
    return environment.getTransactionFactory();
  }
```

获得环境的事务工厂

```java
public TransactionFactory getTransactionFactory() {
    return this.transactionFactory;
  }
```

获取环境的数据源

```java
 public DataSource getDataSource() {
    return this.dataSource;
  }

```

通过事务工厂生成一个事务

```java
// 事务工厂
public interface TransactionFactory {

// JdbcTransaction工厂
public class JdbcTransactionFactory implements TransactionFactory {
    
    
@Override
  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
    return new JdbcTransaction(ds, level, autoCommit);
  }
```

生成一个执行器

```java
// 产生执行器 
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    // 这句再做一下保护，避免有人将defaultExecustorType设为null 
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    // 然后就是简单的三个分支，产生3中执行器BatchExecutor/ReuseExecutor/SimpleExecutor
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    // 如果要求缓存，生成另一种CachingExecutor(默认就是有缓存)，装饰者模式，所以默认都是返回CachingExecutor
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    // 此处调用插件，通过插件可以改变Executor行为
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

调用SimpleExecutor构建器方法

```java
// 执行器
public interface Executor {
    
// 执行器基类
public abstract class BaseExecutor implements Executor {
  //延迟加载队列（线程安全）
  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  //本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询(一级缓存)
  //本地缓存
  protected PerpetualCache localCache;
  //本地输出参数缓存
  protected PerpetualCache localOutputParameterCache;
  protected Configuration configuration;

  //查询堆栈
  protected int queryStack = 0;
  private boolean closed;
    
// 简单执行器
public class SimpleExecutor extends BaseExecutor {
    
    
public SimpleExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
  }
 
protected BaseExecutor(Configuration configuration, Transaction transaction) {
    this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<DeferredLoad>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this;
  }    
```

调用CachingExecutor构造器方法

```java
// 二级缓存执行器
public class CachingExecutor implements Executor {
    
private Executor delegate;
 private TransactionalCacheManager tcm = new TransactionalCacheManager();

  public CachingExecutor(Executor delegate) {
    this.delegate = delegate;
    delegate.setExecutorWrapper(this);
  }
```

会初始化事务缓存管理器

```java
// 事务缓存管理器，被CachingExecutor使用
public class TransactionalCacheManager {
```

调用插件

```java
executor = (Executor) interceptorChain.pluginAll(executor);

// 拦截器链
public class InterceptorChain {
    
    //内部就是一个拦截器的List
  private final List<Interceptor> interceptors = new ArrayList<Interceptor>();
    
    public Object pluginAll(Object target) {
        // 循环调用每个Interceptor.plugin方法
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
    
```

执行器生成结束之后，就通过DefaultSqlSession来创建一个Session

```java
// 这是Mybatis主要的一个类，用来执行SQL,获取映射器，管理事务
// 通常情况下，我们在应用程序中使用的Mybatis的API就是这个接口定义的方法
public interface SqlSession extends Closeable {
   
// 默认SqlSession实现
public class DefaultSqlSession implements SqlSession {
    
    public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
        this.configuration = configuration;
        this.executor = executor;
        this.dirty = false;
        this.autoCommit = autoCommit;
    }
```

## 1.4.通过Session获取映射器

![image-20211203193409653]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211203193409653.png)

```java
@Override
  public <T> T getMapper(Class<T> type) {
      // 最后会去调用MapperRegistry.getMapper
    return configuration.<T>getMapper(type, this);
  }


public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
到映射器注册机获取已经存在的Mapper信息

```java

// 映射器注册机
public class MapperRegistry {
    // 将已经添加的映射都放入HashMap
     private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

// 返回代理类
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

通过映射器代理工厂实例化mapper

```java
// 映射器代理工厂
public class MapperProxyFactory<T> {
      private final Class<T> mapperInterface;
    
    
    public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
    
     protected T newInstance(MapperProxy<T> mapperProxy) {
    //用JDK自带的动态代理生成映射器
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```

## 1.5.通过代理方法执行方法

![image-20211203194621953]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211203194621953.png)

执行方法会调用代理对象的invoke方法

```java

// 映射器代理，代理模式
public class MapperProxy<T> implements InvocationHandler, Serializable {
    
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        // 代理之后，所有Mapper的方法调用时，都会调用这个invoke方法
        // 并不是任何一个方法都需要执行调用代理对象进行执行，如果这个方法是Object中通用的方法（toString、hashCode等）无需执行
        // 判断是否是Object类，如果是它里面的方法直接执行
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
         // 这个没看懂
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
      // 这里优化了，到缓存中找MapperMethod
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

使用cachedMapperMethod(method)在缓存中找

```java
// 去缓存中找MapperMethod
private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
        // 找不到才去new
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```

调用MapperMethod的构造方法

```java
// 映射器方法
public class MapperMethod {

  private final SqlCommand command;
  private final MethodSignature method;
    
   public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, method);
  }  
```

调用SqlCommand的构造方法,它是MapperMethod的静态内部类

```java
// SQL命令，静态内部类
public static class SqlCommand {

    private final String name;
    private final SqlCommandType type;

public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
    // 获取到方法名称
      final String methodName = method.getName();
    // 获取到方法的类名
      final Class<?> declaringClass = method.getDeclaringClass();
    // 解析映射语句
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
      if (ms == null) {
        if (method.getAnnotation(Flush.class) != null) {
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {
        name = ms.getId();
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
    }
    
    
    private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
        Class<?> declaringClass, Configuration configuration) {
        // 通过全限定类名+方法名，得到语句的key
      String statementId = mapperInterface.getName() + "." + methodName;
        // 判断配置中是否有这个语句
        // 如果存在的话
      if (configuration.hasStatement(statementId)) {
          // 从配置中直接得到映射语句
        return configuration.getMappedStatement(statementId);
      } else if (mapperInterface.equals(declaringClass)) {
        return null;
      }
      for (Class<?> superInterface : mapperInterface.getInterfaces()) {
        if (declaringClass.isAssignableFrom(superInterface)) {
          MappedStatement ms = resolveMappedStatement(superInterface, methodName,
              declaringClass, configuration);
          if (ms != null) {
            return ms;
          }
        }
      }
      return null;
    }
  }
```

调用configuration.hasStatement()的方法

```java
public boolean hasStatement(String statementName) {
    return hasStatement(statementName, true);
  }

  public boolean hasStatement(String statementName, boolean validateIncompleteStatements) {
      // 判断是否要进行不完整语句的校验,如果需要的话，要进去校验，并创建
    if (validateIncompleteStatements) {
        // 创建所有语句
      buildAllStatements();
    }
    return mappedStatements.containsKey(statementName);
  }

protected void buildAllStatements() {
    if (!incompleteResultMaps.isEmpty()) {
      synchronized (incompleteResultMaps) {
        // This always throws a BuilderException.
        incompleteResultMaps.iterator().next().resolve();
      }
    }
    if (!incompleteCacheRefs.isEmpty()) {
      synchronized (incompleteCacheRefs) {
        // This always throws a BuilderException.
        incompleteCacheRefs.iterator().next().resolveCacheRef();
      }
    }
    if (!incompleteStatements.isEmpty()) {
      synchronized (incompleteStatements) {
        // This always throws a BuilderException.
        incompleteStatements.iterator().next().parseStatementNode();
      }
    }
    if (!incompleteMethods.isEmpty()) {
      synchronized (incompleteMethods) {
        // This always throws a BuilderException.
        incompleteMethods.iterator().next().resolve();
      }
    }
  }

```

得到映射语句之后，进行判断

```java
	if (ms == null) {
        if (method.getAnnotation(Flush.class) != null) {
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {
        // 获取对应的key
        name = ms.getId();
        // 获取SQL命令的类型
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
```

初始化SqlCommand后，接着初始化MethodSignature

```java
// 方法签名，静态内部类 
public static class MethodSignature {

    private final boolean returnsMany;
    private final boolean returnsMap;
    private final boolean returnsVoid;
    private final boolean returnsCursor;
    private final Class<?> returnType;
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      if (resolvedReturnType instanceof Class<?>) {
          // 将解析后的结果，赋值
        this.returnType = (Class<?>) resolvedReturnType;
      } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
        // 这些都不重要
      this.returnsVoid = void.class.equals(this.returnType);
      this.returnsMany = (configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray());
      this.returnsCursor = Cursor.class.equals(this.returnType);
      this.mapKey = getMapKey(method);
      this.returnsMap = (this.mapKey != null);
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
       // 注解名称解析器，可以直接忽略
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
```

到类型参数解析器里解析返回类型(并不重要)

```java
public class TypeParameterResolver {
    
    public static Type resolveReturnType(Method method, Type srcType) {
    Type returnType = method.getGenericReturnType();
    Class<?> declaringClass = method.getDeclaringClass();
    return resolveType(returnType, srcType, declaringClass);
  }

    
     private static Type resolveType(Type type, Type srcType, Class<?> declaringClass) {
    if (type instanceof TypeVariable) {
      return resolveTypeVar((TypeVariable<?>) type, srcType, declaringClass);
    } else if (type instanceof ParameterizedType) {
      return resolveParameterizedType((ParameterizedType) type, srcType, declaringClass);
    } else if (type instanceof GenericArrayType) {
      return resolveGenericArrayType((GenericArrayType) type, srcType, declaringClass);
    } else {
        // 从这里返回
      return type;
    }
  }
```

初始化MappedMethod之后，把它放在缓存中

```java
//找不到才去new
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
```

创建结束后，去执行

```java
//这里优化了，去缓存中找MapperMethod
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //执行
    return mapperMethod.execute(sqlSession, args);
```

MapperMethod中的值

![image-20211204155314402]( https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20211204155314402.png)

进行执行

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    // 可以看到执行时就是4种情况，insert|update|delete|select,分别调用SqlSession的4大类方法
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) { // 如果有结果处理器
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) { // 如果结果有多多条记录
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) { // 如果结果是map
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else { // 否则就是一条记录
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

转换参数为SQL命令中的参数

```java
public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
        // 如果没参数
      return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
        // 如果只有一个参数
      return args[names.firstKey()];
    } else {
        // 否则，返回一个ParamMap，修改参数名，参数名就是其位置
      final Map<String, Object> param = new ParamMap<Object>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
          // 1.先加一个#{0},#{1},#{2}...参数
        param.put(entry.getValue(), args[entry.getKey()]);
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);  //  GENERIC_NAME_PREFIX = "param"
        if (!names.containsValue(genericParamName)) {
            // 2.再加一个#{param1},#{param2}...参数
            // 你可以传递多个参数给映射器方法，如果你这样做了
            // 默认情况下它们将会以它们在参数列表中的位置来命名，比如：#{param1},#{param2}等.
            // 如果你想改变参数的名称（只在多参数情况下），那么你可以在参数上使用@Param("paramName")注解。
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
```

执行方法

```java
  Object param = method.convertArgsToSqlCommandParam(args);
   result = sqlSession.selectOne(command.getName(), param);
```

调用selectOne方法

```java
// 核心selectionOne
@Override
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
      // 转而去调用selectList，很简单的，如果得到0条则返回null，得到1条则返回1条，得到多条报TooManyResultsException错
      // 特别需要注意的是当没有查询到结果的时候，就会返回null。因此一般建议在mapper中编写resultType的时候使用包装类型
      // 而不是基本类型，比如推荐使用Integer而不是int，这样就可以避免NPE
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }
```

调用selectList方法

```java
@Override
  public <E> List<E> selectList(String statement, Object parameter) {
      // 这里传入了一个RowBounds.DEFAULT
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
  }


// 核心selectList
@Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // 根据statement id找到对应的MappedStatement
        // 进去会继续检查是否有不完整的语句，进行创建，检查过后，直接返回
      MappedStatement ms = configuration.getMappedStatement(statement);
        // 转而用执行器来查询结果，注意这里传入的ResultHandler是null
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

// 把参数包装成Collection
private Object wrapCollection(final Object object) {
    if (object instanceof Collection) {
        // 参数若是Collection型，做collection标记
      StrictMap<Object> map = new StrictMap<Object>();
      map.put("collection", object);
      if (object instanceof List) {
          // 参数若是List型，做list标记
        map.put("list", object);
      }
      return map;
    } else if (object != null && object.getClass().isArray()) {
        // 参数若是数组型，做array标记
      StrictMap<Object> map = new StrictMap<Object>();
      map.put("array", object);
      return map;
    }
    // 参数若不是集合型，直接返回原来值
    return object;
  }
```

RowBounds类

```java
// 分页用，记录限制
public class RowBounds {

  public static final int NO_ROW_OFFSET = 0;
  public static final int NO_ROW_LIMIT = Integer.MAX_VALUE;
  public static final RowBounds DEFAULT = new RowBounds();
    
    // 默认是一页Integer.MAX_VALUE条
    public RowBounds() {
    this.offset = NO_ROW_OFFSET;
    this.limit = NO_ROW_LIMIT;
  }
```

调用缓存执行器的query方法

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
      // query时传入一个cachekey参数
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

> MapperMethod类为 映射器方法
>
> MappedStatement类为 映射的语句

调用getBoundSql方法

```java
public BoundSql getBoundSql(Object parameterObject) {
    // 其实就是调用sqlSource.getBoundSql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 剩下的可以暂时忽略
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }
```

调用RawSqlSource的getBoundSql方法

```java
// 原始SQL源码，比DynamicSqlSource快
public class RawSqlSource implements SqlSource {

@Override
  public BoundSql getBoundSql(Object parameterObject) {
    return sqlSource.getBoundSql(parameterObject);
  }
```

接着又调用了StaticSqlSource的getBoundSql方法

```java
// 静态SQL源码
public class StaticSqlSource implements SqlSource {
    
    @Override
  public BoundSql getBoundSql(Object parameterObject) {
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
  }
```

调用BoundSql的构造方法

```java
// 绑定的SQL，是从SqlSource而来，将动态内容都处理完成得到的SQL语句字符串，其中包括?,还有绑定的参数
public class BoundSql {
    
public BoundSql(Configuration configuration, String sql, List<ParameterMapping> parameterMappings, Object parameterObject) {
    this.sql = sql;
    this.parameterMappings = parameterMappings;
    this.parameterObject = parameterObject;
    this.additionalParameters = new HashMap<String, Object>();
    this.metaParameters = configuration.newMetaObject(additionalParameters);
  }
```

创建成功后，创建一个CacheKey

```java
// query时传入一个cachekey参数
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
```

调用BaseExecutor的方法

```java
// 创建缓存可以
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
      // Mybatis 对于其Key 的生成采取规则为:[mappedStatmentId + offset + limit + SQL + queryParams + environment]生成一个哈希码
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // 模仿DefaultParameterHandler的逻辑，不能重复，请参考DefaultParameterHandler
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

缓存key类

```java
// 缓存类
// 一般缓存框架的数据结构基本上都是 Key-Value方法存储
public class CacheKey implements Cloneable, Serializable {
    
     public CacheKey() {
    this.hashcode = DEFAULT_HASHCODE;
    this.multiplier = DEFAULT_MULTIPLYER;
    this.count = 0;
    this.updateList = new ArrayList<Object>();
  }
```

执行query

```java
// 被ResultLoader.selectList调用
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
      // 默认情况下是没有开启缓存的（二级缓存），要开启二级缓存，你需要在你的SQL映射文件中添加一行:<cache/>
      // 简单的说，就是先查CacheKey,查不到再委托给实际的执行器去查
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
```

通过BaseExecutor.query方法去查

```java
 @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    // 如果已经关闭，报错
      if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
      // 先清局部缓存，再查询，但仅查询堆栈为0，才清。为了处理递归调用
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
        // 加一，这样递归调用到上面的时候就不会再清局部缓存了
      queryStack++;
        // 先根据cacheKey从localCache去查
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
          // 若查到localCache缓存，处理localOutPutParameterCache
          // 此时缓存中是没有值的，因此要从数据库中去查
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
          // 从数据库查
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
        // 清空堆栈
      queryStack--;
    }
    if (queryStack == 0) {
        // 延迟加载队列中所有元素
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
        // 清空延迟加载队列
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // 如果是STATEMENT，清本地缓存
        clearLocalCache();
      }
    }
    return list;
  }
```

从数据库中查

```java
// 从数据库中查
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 先向缓存中放入占位符
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // 最后删除占位符
      localCache.removeObject(key);
    }
    // 加入缓存
    localCache.putObject(key, list);
    // 如果是存储过程，OUT参数也加入缓存
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }

```

调用PerpetualCache的方法

```java
// 缓存
public interface Cache {

// 永久缓存
// 一旦存入就一直保持
public class PerpetualCache implements Cache {

    // 每个永久缓存有一个ID来识别
  private final String id;

    // 内部就是一个HashMap，所有方法就是基本调用HashMap的方法，不支持多线程?
  private Map<Object, Object> cache = new HashMap<Object, Object>();
    
    @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }
```

EXECUTION_PLACEHOLDER是一个枚举类的一个属性

```java
// 执行占位符
public enum ExecutionPlaceholder {
  EXECUTION_PLACEHOLDER
}
```

调用SimpleExecutor.doQuery（）方法

```java
@Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      // 新建一个StatementHandler
        // 这里看到ResultHandler传入了
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
     // 准备语句
        stmt = prepareStatement(handler, ms.getStatementLog());
        // StatementHandler.query
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

调用Configuration里的创建StatementHandler方法

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 创建路由选择语句处理器
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 插件在这里插入
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
```

使用路由选择语句处理器的构造方法

```java
// 语句处理器
public interface StatementHandler {
    
// 路由选择语句处理器，有点像代理模式    
public class RoutingStatementHandler implements StatementHandler {

  private final StatementHandler delegate;

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

      // 根据语句类型，委派到不同的语句处理器(STATEMENT|PREPARED|CALLABLE)
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
```

经过插件的责任链之后，语句处理器创建结束，进行准备语句

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    // 调用StatementHandler.prepare
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 调用StatementHandler.parameterize
    handler.parameterize(stmt);
    return stmt;
  }
```

调用BaseExecutor的getConnection方法

```java
protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
      	// 如果需要打印Connection的日志，返回一个ConnectionLogger(代理模式，AOP思想)
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
```

从事务中获得一个连接

```java
// 事务，包装了一个Connection，包含commit，rollback，close方法
// 在Mybatis中有两个事务管理器类型(也就是 type = "[JDBC|MANAGED]")
public interface Transaction {
   
// Jdbc事务。直接利用JDBC的commit，rollback
// 它依赖于数据源得到的连接来管理事务范围
public class JdbcTransaction implements Transaction {   
    
    @Override
  public Connection getConnection() throws SQLException {
    if (connection == null) {
        // 开启一个连接
      openConnection();
    }
    return connection;
  }
    
   protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    connection = dataSource.getConnection();
    if (level != null) {
      connection.setTransactionIsolation(level.getLevel());
    }
    setDesiredAutoCommit(autoCommmit);
  }

  protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
    try {
        // 和原来的比一下，再设置autocommit，是考虑多次重复设置的性能问题?
      if (connection.getAutoCommit() != desiredAutoCommit) {
        if (log.isDebugEnabled()) {
          log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + connection + "]");
        }
        connection.setAutoCommit(desiredAutoCommit);
      }
    } catch (SQLException e) {
      // Only a very poorly implemented driver would fail here,
      // and there's not much we can do about that.
      throw new TransactionException("Error configuring AutoCommit.  "
          + "Your driver may not support getAutoCommit() or setAutoCommit(). "
          + "Requested setting: " + desiredAutoCommit + ".  Cause: " + e, e);
    }
  }
```

调用StatementHandler.prepare

```java
// 先调用RoutingStatementHandler的prepare方法
@Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    return delegate.prepare(connection, transactionTimeout);
  }

```

再调用BaseStatementHandler的prepare方法

```java
@Override
  public Statement prepare(Connection connection) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      //实例化Statement
      statement = instantiateStatement(connection);
      //设置超时
      setStatementTimeout(statement);
      //设置读取条数
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }

// 如何实例化Statement，交给子类做
protected abstract Statement instantiateStatement(Connection connection) throws SQLException;

```

调用PreparedStatementHandler的方法

```java
@Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    //调用Connection.prepareStatement
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.prepareStatement(sql);
    }
  }
```

实例化Statement之后，设置超时时间

```java
// 设置超时时间，其实就是调用Statement.setQueryTimeout
protected void setStatementTimeout(Statement stmt) throws SQLException {
    Integer timeout = mappedStatement.getTimeout();
    Integer defaultTimeout = configuration.getDefaultStatementTimeout();
    if (timeout != null) {
      stmt.setQueryTimeout(timeout);
    } else if (defaultTimeout != null) {
      stmt.setQueryTimeout(defaultTimeout);
    }
  }
```

接着设置读取条数

```java
// 设置读取条数，其实就是调用Statement,setFetchSize
protected void setFetchSize(Statement stmt) throws SQLException {
    Integer fetchSize = mappedStatement.getFetchSize();
    if (fetchSize != null) {
      stmt.setFetchSize(fetchSize);
    }
  }
```

准备语句结束之后，调用参数化方法

```java
// 先调用路由语句处理器的方法
@Override
  public void parameterize(Statement statement) throws SQLException {
    delegate.parameterize(statement);
  }

// 再调用预处理语句处理器的方法
public void parameterize(Statement statement) throws SQLException {
    //调用ParameterHandler.setParameters
    parameterHandler.setParameters((PreparedStatement) statement);
  }
```

调用参数处理器的方法

```java
// 参数处理器
public interface ParameterHandler {

// 默认参数处理器
public class DefaultParameterHandler implements ParameterHandler {
    
    
    // 设置参数
    @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
        // 循环设参数
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          // 如果不是OUT，才设进去
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            // 若有额外的参数，设为额外的参数
              value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
              // 若参数为null，直接设null
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {  // 调用这个
              // 若参数有对应的TypeHandler，直接设object
            value = parameterObject;
          } else {
              // 除此以外，MetaObject.getValue反射取得值设进去
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
              // 不同类型的set方法不同，所以委派给子类的setParameter方法
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
```

调用BaseTypeHandler.setParameter()

```java
@Override
  public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
      // 特殊情况，设置NULL
    if (parameter == null) {
      if (jdbcType == null) {
          // 如果没设置jdbcType，报错了
        throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
      }
      try {
          // 设成NULL
        ps.setNull(i, jdbcType.TYPE_CODE);
      } catch (SQLException e) {
        throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + " . " +
                "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. " +
                "Cause: " + e, e);
      }
    } else {
      try {
          // 非NULL情况，怎么还设得给交给不同的子类完成，setNonNullParameter是一个抽象方法
        setNonNullParameter(ps, i, parameter, jdbcType);
      } catch (Exception e) {
        throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + " . " +
                "Try setting a different JdbcType for this parameter or a different configuration property. " +
                "Cause: " + e, e);
      }
    }
  }
```

由于mapper.xml中的resultType并没有设置，因此没有jdbcType

```java
 @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType)
      throws SQLException {
    TypeHandler handler = resolveTypeHandler(parameter, jdbcType);
    handler.setParameter(ps, i, parameter, jdbcType);
  }


 private TypeHandler<? extends Object> resolveTypeHandler(Object parameter, JdbcType jdbcType) {
    TypeHandler<? extends Object> handler;
    if (parameter == null) {
      handler = OBJECT_TYPE_HANDLER;
    } else {
      handler = typeHandlerRegistry.getTypeHandler(parameter.getClass(), jdbcType);
      // check if handler is null (issue #270)
      if (handler == null || handler instanceof UnknownTypeHandler) {
        handler = OBJECT_TYPE_HANDLER;
      }
    }
    return handler;
  }
```

会通过参数获得其typeHandler再调用其方法进行设置

```java
@Override
  public void setNonNullParameter(PreparedStatement ps, int i, Integer parameter, JdbcType jdbcType)
      throws SQLException {
    ps.setInt(i, parameter);
  }
```

得到PreparedStatement之后，回来调用SimpleExecutor中的语句处理器的query方法

```java
stmt = prepareStatement(handler, ms.getStatementLog());
 return handler.<E>query(stmt, resultHandler);
```

代理到PreparedStatementHandler执行方法

```java
@Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleResultSets(ps);
  }
```

再到DefaultResultSetHandler中处理结果集

```java
 @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
    
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    //一般resultMaps里只有一个元素
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResulSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }



// 处理结果集
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
            // 如果没有resultHandler
            // 新建DefaultResultHandler
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
            // 调用自己的handleRowValues
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
            // 得到记录的list
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
            // 如果有resultHandler
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
        // 最后别忘了关闭结果集
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }
```

处理完结果集之后，再到DefaultResultSetHandler去执行collapseSingleResultList

```java
private List<Object> collapseSingleResultList(List<Object> multipleResults) {
    return multipleResults.size() == 1 ? (List<Object>) multipleResults.get(0) : multipleResults;
  }
```



