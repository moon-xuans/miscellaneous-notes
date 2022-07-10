## AOP源码剖析

## 1.前言

要在spring中开启AOP功能，还需要在配置文件中添加以下声明

<aop:aspectj-autoproxy />表示spring支持注解的AOP

 

在init方法中，**一旦遇到<aop:aspectj-autoproxy />注解时就会使用解析器AspectJAutoProxyBeanDefinitionParser进行解析**

```java
Public void init(){

......

registerBeanDefinitionParser(“aspectj-autoproxy”,new AspectJAutoProxyBeanDefinitionParser());

......

}
```

**所有的解析器**，都是对BeanDefinitionParser接口的实现，**入口都是从parser函数开始的**

AspectJAutoProxyBeanDefinitionParser的parser方法：

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
        				    AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
        this.extendBeanDefinition(element, parserContext);
        return null;
    }
```

## 2.重点

```java
public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(ParserContext parserContext, Element sourceElement) {
    // 注册或者升级自动代理创建器(调用registerAspectJAnnotationAutoProxyCreatorIfNecesssary方法)
    BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext.getRegistry(), parserContext.extractSource(sourceElement));
    // 调用useClassProxyingIfNexessary方法，对于proxy-target-class以及expose-proxy属性的处理
    useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
    registerComponentIfNecessary(beanDefinition, parserContext);
}
```

### 2.1.调用registerAspectJAnnotationAutoProxyCreatorIfNecesssary方法

**对于AOP的实现，基本上都是靠自动代理创建器去完成，它可以根据@Point注解定义的切点来自动代理相匹配的bean **

```java
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, @Nullable Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

 @Nullable
    private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
            BeanDefinition apcDefinition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
            if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
                int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
                int requiredPriority = findPriorityForClass(cls);
                // 1.如果已经存在了自动代理创建器，而且存在的自动代理器与现在的不一致，那么需要根据优先级判断到底使用哪一个
                if (currentPriority < requiredPriority) {
                    apcDefinition.setBeanClassName(cls.getName());
                }
            }
			// 2.如果存在自动代理创建器并且与将要的一致，就无需创建，返回空值
            return null;
        } else {
            // 如果不存在自动代理器，则在在这里创建一个
            RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
            beanDefinition.setSource(source);
            beanDefinition.getPropertyValues().add("order", -2147483648);
            beanDefinition.setRole(2);
            registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator", beanDefinition);
            return beanDefinition;
        }
    }
```

### 2.2.调用useClassProxyingIfNexessary

**怎么处理？useClassProxyingIfNexessary方法**

proxy-target-class：springAop部分使用JDK动态代理或者CGLIB来为目标对象创建代理（建议尽量使用JDK的动态代理），如果被代理的目标对象实现至少一个接口，则会使用JDK动态代理，所有该目标类型实现的接口都将被代理，若该目标对象没有实现任何接口，则创建一个CGLIB代理

如果想要强制使用CGLIB动态代理需要将<aop:config>属性的proxy-target-class设置为true

当需要使用ＣＧＬＩＢ代理和＠AspectJ自动代理支持，可以按照以下方式设置＜ａｏｐ：ａｓｐｅｃｔＪ－ａｕｔｏｐｒｏｘｙ　ｐｒｏｘｙ－ｔａｒｇｅｔ－ｃｌａｓｓ＝＂ｔｒｕｅ＂＞

expose-proxy:有时候目标对象内部的自我调节将无法实施切面中的增强

```java
 private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, @Nullable Element sourceElement) {
        if (sourceElement != null) {
            // 1.对proxyTargetClass进行判断
            boolean proxyTargetClass = Boolean.parseBoolean(sourceElement.getAttribute("proxy-target-class"));
            if (proxyTargetClass) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }

            // 2.对exposeProxy进行判断
            boolean exposeProxy = Boolean.parseBoolean(sourceElement.getAttribute("expose-proxy"));
            if (exposeProxy) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }

    }
```

### 2.3.创建AOP代理

上面讲了自定义配置完成了自动代理创建器的自动注册，那么这个类到底做了什么来完成AOP?

自动代理创建器实现了BeanPostProcessor接口，实现了这个接口以后，**当spring加载这个Bean时会在实例化前调用其postProcessAfterInitialization方法**

在父类AbstractAutoProxyCreator的postProcessAfterInitialization方法如下：

####  2.3.1.getCacheKey

```java
 public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
        if (bean != null) {
            // 根据给定的bean的class和name构建出一个key,格式beanClassName_beanName,调用wrapIfNecessary方法（如果它适合被代理，就需要封装指定的bean）
            Object cacheKey = this.getCacheKey(bean.getClass(), beanName);
            if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                return this.wrapIfNecessary(bean, beanName, cacheKey);
            }
        }

        return bean;
    }


```

#### 2.3.2.wrapIfNecessary方法

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 1.如果这个bean已经处理过，无需增强，判断给定的bean类是否代表一个基础设施类，基础设施类不应代理，或者配置了指定bean不需要被代理直接返回bean
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    } else if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    } else if (!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) {
        // 2.通过调用getAdvicesAndAdvisorsForBean方法,如果需要增强就获取增强方法或增强器
        Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
        // 3.如果获取到了增强则需要针对增强创建代理
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        } else {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
    } else {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
}
```

##### 2.3.2.2.getAdvicesAndAdvisorsForBean

getAdvicesAndAdvisorsForBean调用findCandidateAdvisors方法获取所有的增强，调用	findAdvisorsThatCanApply遍历所有增强中适用于bean的增强并应用

```java
 @Nullable
    protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
        List<Advisor> advisors = this.findEligibleAdvisors(beanClass, beanName);
        return advisors.isEmpty() ? DO_NOT_PROXY : advisors.toArray();
    }
```
```java
 protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
     	// 1.调用findCandidateAdvisors方法获取所有的增强
        List<Advisor> candidateAdvisors = this.findCandidateAdvisors();
     	// 2.调用findAdvisorsThatCanApply遍历所有增强中适用于bean的增强并应用
        List<Advisor> eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
        this.extendAdvisors(eligibleAdvisors);
        if (!eligibleAdvisors.isEmpty()) {
            eligibleAdvisors = this.sortAdvisors(eligibleAdvisors);
        }

        return eligibleAdvisors;
    }
```
```java
// findCandidateAdvisors方法调用buildAspectJAdvisors方法，buildAspectJAdvisor方法中 
protected List<Advisor> findCandidateAdvisors() {
        List<Advisor> advisors = super.findCandidateAdvisors();
        if (this.aspectJAdvisorsBuilder != null) {
            advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
        }

        return advisors;
    }

 public List<Advisor> buildAspectJAdvisors() {
        List<String> aspectNames = this.aspectBeanNames;
        if (aspectNames == null) {
            synchronized(this) {
                aspectNames = this.aspectBeanNames;
                if (aspectNames == null) {
                    List<Advisor> advisors = new ArrayList();
                    List<String> aspectNames = new ArrayList();
                    // 1.获取所有的beanName,将所有在beanFactory中注册的bean都会被提取出来
                    String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false);
                    String[] var18 = beanNames;
                    int var19 = beanNames.length;

                    for(int var7 = 0; var7 < var19; ++var7) {
                        String beanName = var18[var7];
                        if (this.isEligibleBean(beanName)) {
                            Class<?> beanType = this.beanFactory.getType(beanName, false);
                            // 2.遍历所有的beanName,并找出声明AspectJ注解的类
                            if (beanType != null && this.advisorFactory.isAspect(beanType)) {
                                aspectNames.add(beanName);
                                AspectMetadata amd = new AspectMetadata(beanType, beanName);
                                if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                                    MetadataAwareAspectInstanceFactory factory = new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                                    // 3.对标记为AspectJ注解的类进行增强器的提取，通过调用getAdvisors方法,getAdvisors调用getAdvisor方法
                                    List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                                    // 4.将提取的结果加入缓存
                                    if (this.beanFactory.isSingleton(beanName)) {
                                        this.advisorsCache.put(beanName, classAdvisors);
                                    } else {
                                        this.aspectFactoryCache.put(beanName, factory);
                                    }

                                    advisors.addAll(classAdvisors);
                                } else {
                                    if (this.beanFactory.isSingleton(beanName)) {
                                        throw new IllegalArgumentException("Bean with name '" + beanName + "' is a singleton, but aspect instantiation model is not singleton");
                                    }

                                    MetadataAwareAspectInstanceFactory factory = new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                                    this.aspectFactoryCache.put(beanName, factory);
                                    advisors.addAll(this.advisorFactory.getAdvisors(factory));
                                }
                            }
                        }
                    }

                    this.aspectBeanNames = aspectNames;
                    return advisors;
                }
            }
        }

        if (aspectNames.isEmpty()) {
            return Collections.emptyList();
        } else {
            List<Advisor> advisors = new ArrayList();
            Iterator var3 = aspectNames.iterator();

            while(var3.hasNext()) {
                String aspectName = (String)var3.next();
                List<Advisor> cachedAdvisors = (List)this.advisorsCache.get(aspectName);
                if (cachedAdvisors != null) {
                    advisors.addAll(cachedAdvisors);
                } else {
                    MetadataAwareAspectInstanceFactory factory = (MetadataAwareAspectInstanceFactory)this.aspectFactoryCache.get(aspectName);
                    advisors.addAll(this.advisorFactory.getAdvisors(factory));
                }
            }

            return advisors;
        }
    }
```

###### 2.3.2.2.3.getAdvisor方法

getAdvisors调用getAdvisor方法

```java
 public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
        Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
        String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
        this.validate(aspectClass);
        MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory = new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
        List<Advisor> advisors = new ArrayList();
        Iterator var6 = this.getAdvisorMethods(aspectClass).iterator();

        while(var6.hasNext()) {
            Method method = (Method)var6.next();
            Advisor advisor = this.getAdvisor(method, lazySingletonAspectInstanceFactory, 0, aspectName);
            if (advisor != null) {
                advisors.add(advisor);
            }
        }

        if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
            Advisor instantiationAdvisor = new ReflectiveAspectJAdvisorFactory.SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
            advisors.add(0, instantiationAdvisor);
        }

        Field[] var12 = aspectClass.getDeclaredFields();
        int var13 = var12.length;

        for(int var14 = 0; var14 < var13; ++var14) {
            Field field = var12[var14];
            Advisor advisor = this.getDeclareParentsAdvisor(field);
            if (advisor != null) {
                advisors.add(advisor);
            }
        }

        return advisors;
    }


  @Nullable
    public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrderInAspect, String aspectName) {
        this.validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
        // 1.调用getPointcut方法获取切点信息
        // 获取方法上的注解并封装
        AspectJExpressionPointcut expressionPointcut = this.getPointcut(candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
        // 2.根据切点信息生成增强器，通过new InstantiationModelPointcutAdvisorImpl()
        // spring根据不同的注解生成不同的增强器
        return expressionPointcut == null ? null : new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod, this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
    }

```

##### 2.3.2.3.createProxy方法

 createProxy委托给ProxyFactory去对于代理类的创建及处理，createProxy主要对ProxyFactory进行一些初始化操作

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName, @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        // 1.获取当前类的属性
    AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory)this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    if (!proxyFactory.isProxyTargetClass()) {
        if (this.shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        } else {
            // 2.添加代理接口
            this.evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 3.增强封装器
    Advisor[] advisors = this.buildAdvisors(beanName, specificInterceptors);
    // 4.将增强器放入代理创建工厂中
    proxyFactory.addAdvisors(advisors);
    // 5.设置要代理的类
    proxyFactory.setTargetSource(targetSource);
    // 6.定制代理（在spring中为子类提供了定制函数customizeProxyFactory,子类可以在此函数中进行ProxyFactory的进一步封装）
    this.customizeProxyFactory(proxyFactory);
    proxyFactory.setFrozen(this.freezeProxy);
    if (this.advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    ClassLoader classLoader = this.getProxyClassLoader();
    if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {
        classLoader = ((SmartClassLoader)classLoader).getOriginalClassLoader();
    }

    // 7.进行获取代理操作( proxyFactory.getProxy()->createAopProxy().getProxy()，createAopProxy()->getAopProxyFactory().createAopProxy() )
    return proxyFactory.getProxy(classLoader);
}
```

###### 2.3.2.3.3.buildAdvisors方法

```java
 protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
        Advisor[] commonInterceptors = this.resolveInterceptorNames();
        List<Object> allInterceptors = new ArrayList();
        if (specificInterceptors != null) {
            if (specificInterceptors.length > 0) {
                allInterceptors.addAll(Arrays.asList(specificInterceptors));
            }

            if (commonInterceptors.length > 0) {
                if (this.applyCommonInterceptorsFirst) {
                    allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
                } else {
                    allInterceptors.addAll(Arrays.asList(commonInterceptors));
                }
            }
        }

        int i;
        if (this.logger.isTraceEnabled()) {
            int nrOfCommonInterceptors = commonInterceptors.length;
            i = specificInterceptors != null ? specificInterceptors.length : 0;
            this.logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors + " common interceptors and " + i + " specific interceptors");
        }

        Advisor[] advisors = new Advisor[allInterceptors.size()];

        for(i = 0; i < allInterceptors.size(); ++i) {
            // 这里调用wrap方法
            advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
        }

        return advisors;
    }


 public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
        if (adviceObject instanceof Advisor) { // 1.如果要封装的对象是增强器就不用处理
            return (Advisor)adviceObject;
        } else if (!(adviceObject instanceof Advice)) { // 2.如果要封装的对象不是advisor也不是advice类型就抛异常
            throw new UnknownAdviceTypeException(adviceObject);
        } else {
            Advice advice = (Advice)adviceObject;
            if (advice instanceof MethodInterceptor) { // 3.如果是MethodInceptor就封装为相应的增强器
                return new DefaultPointcutAdvisor(advice);
            } else { // 4.如果存在增强器的适配器也要进行封装
                Iterator var3 = this.adapters.iterator();

                AdvisorAdapter adapter;
                do {
                    if (!var3.hasNext()) {
                        throw new UnknownAdviceTypeException(advice);
                    }

                    adapter = (AdvisorAdapter)var3.next();
                } while(!adapter.supportsAdvice(advice));

                return new DefaultPointcutAdvisor(advice);
            }
        }
    }
```

###### 2.3.2.3.7. proxyFactory.getProxy(classLoader)

proxyFactory.getProxy()->createAopProxy().getProxy()，createAopProxy()->getAopProxyFactory().createAopProxy() 

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return this.createAopProxy().getProxy(classLoader);
}

```

**createAopProxy()**

```java
 protected final synchronized AopProxy createAopProxy() {
        if (!this.active) {
            this.activate();
        }

        return this.getAopProxyFactory().createAopProxy(this);
    }


    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
       	// 1.如果采用激进的优化策略或者目标类本身被代理或者不存在代理接口
        // 1.1.如果目标类本身是一个接口，则采用jdk动态代理
        // 1.2.否则采用CGLIB
        // 2.否则采用JDK代理
        if (NativeDetector.inNativeImage() || !config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
            return new JdkDynamicAopProxy(config); // 采用CGLIB
        } else {
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
            } else {
                return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
            }
        }
    }
```

**getProxy()**

**①JDK代理**

**JDKProxy**实现了InvocationHandler接口，那么就会有一个invoke函数

**Invoke方法**

主要是**创建一个拦截器链**，调用ReflectiveMethodInvocation的proceed方法**逐一调用拦截器**，**执行完所有的增强后调用切点方法**

```java

  public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isTraceEnabled()) {
            logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
        }

        return Proxy.newProxyInstance(classLoader, this.proxiedInterfaces, this);
    }


 @Nullable
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;
        TargetSource targetSource = this.advised.targetSource;
        Object target = null;

        Boolean var8;
        try {
            if (this.equalsDefined || !AopUtils.isEqualsMethod(method)) {
                if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                    Integer var18 = this.hashCode();
                    return var18;
                }

                if (method.getDeclaringClass() == DecoratingProxy.class) {
                    Class var17 = AopProxyUtils.ultimateTargetClass(this.advised);
                    return var17;
                }

                Object retVal;
                if (!this.advised.opaque && method.getDeclaringClass().isInterface() && method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                    retVal = AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
                    return retVal;
                }

                if (this.advised.exposeProxy) {
                    oldProxy = AopContext.setCurrentProxy(proxy);
                    setProxyContext = true;
                }

                target = targetSource.getTarget();
                Class<?> targetClass = target != null ? target.getClass() : null;
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                if (chain.isEmpty()) {
                    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
                } else {
                    // 1.创建一个拦截器链
                    MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                    // 2.调用proceed方法逐一调用拦截器，执行完所有的增强后调用切点方法
                    retVal = invocation.proceed();
                }

                Class<?> returnType = method.getReturnType();
                if (retVal != null && retVal == target && returnType != Object.class && returnType.isInstance(proxy) && !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                    retVal = proxy;
                } else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                    throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
                }

                Object var12 = retVal;
                return var12;
            }

            var8 = this.equals(args[0]);
        } finally {
            if (target != null && !targetSource.isStatic()) {
                targetSource.releaseTarget(target);
            }

            if (setProxyContext) {
                AopContext.setCurrentProxy(oldProxy);
            }

        }

        return var8;
    }
```

**②CGLIBProxy**

**入口是getProxy方法**

CglibMethodInvocation继承ReflectiveMethodInvocation

通过getCallbacks方法**设置拦截器链**，**将拦截器链加入callback,加入callback后，调用代理时会直接调用**DynamicAdvisedInterceptor的**intercept方法**

intercept方法

**获取拦截器链，如果为空直接执行原方法，否则进入拦截器链**通过ReflectiveMethodInvocation的proceed方法**挨个调用拦截器**

```java
 public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isTraceEnabled()) {
            logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
        }

        try {
            Class<?> rootClass = this.advised.getTargetClass();
            Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");
            Class<?> proxySuperClass = rootClass;
            int x;
            if (rootClass.getName().contains("$$")) {
                proxySuperClass = rootClass.getSuperclass();
                Class<?>[] additionalInterfaces = rootClass.getInterfaces();
                Class[] var5 = additionalInterfaces;
                int var6 = additionalInterfaces.length;

                for(x = 0; x < var6; ++x) {
                    Class<?> additionalInterface = var5[x];
                    this.advised.addInterface(additionalInterface);
                }
            }

            this.validateClassIfNecessary(proxySuperClass, classLoader);
            Enhancer enhancer = this.createEnhancer();
            if (classLoader != null) {
                enhancer.setClassLoader(classLoader);
                if (classLoader instanceof SmartClassLoader && ((SmartClassLoader)classLoader).isClassReloadable(proxySuperClass)) {
                    enhancer.setUseCache(false);
                }
            }

            enhancer.setSuperclass(proxySuperClass);
            enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
            enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
            enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));
            // 通过getCallbacks方法设置拦截器链
            Callback[] callbacks = this.getCallbacks(rootClass);
            Class<?>[] types = new Class[callbacks.length];

            for(x = 0; x < types.length; ++x) {
                types[x] = callbacks[x].getClass();
            }

            enhancer.setCallbackFilter(new CglibAopProxy.ProxyCallbackFilter(this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
            enhancer.setCallbackTypes(types);
            return this.createProxyClassAndInstance(enhancer, callbacks);
        } catch (IllegalArgumentException | CodeGenerationException var9) {
            throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() + ": Common causes of this problem include using a final class or a non-visible class", var9);
        } catch (Throwable var10) {
            throw new AopConfigException("Unexpected AOP exception", var10);
        }
    }


private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

    @Nullable
            public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
                Object oldProxy = null;
                boolean setProxyContext = false;
                Object target = null;
                TargetSource targetSource = this.advised.getTargetSource();

                Object var16;
                try {
                    if (this.advised.exposeProxy) {
                        oldProxy = AopContext.setCurrentProxy(proxy);
                        setProxyContext = true;
                    }

                    target = targetSource.getTarget();
                    Class<?> targetClass = target != null ? target.getClass() : null;
                    // 1.获取拦截器链
                    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                    Object retVal;
                    if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
                        Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                        // 2.如果为空，执行原方法
                        retVal = methodProxy.invoke(target, argsToUse);
                    } else {
                        // 否则进入拦截器链通过ReflectiveMethodInvocation的proceed方法挨个调用拦截器
                        retVal = (new CglibAopProxy.CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy)).proceed();
                    }

                    retVal = CglibAopProxy.processReturnType(proxy, target, method, retVal);
                    var16 = retVal;
                } finally {
                    if (target != null && !targetSource.isStatic()) {
                        targetSource.releaseTarget(target);
                    }

                    if (setProxyContext) {
                        AopContext.setCurrentProxy(oldProxy);
                    }

                }

                return var16;
            }
}
```

