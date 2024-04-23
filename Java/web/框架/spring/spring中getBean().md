# spring中getBean()

首先，Bean的加载调用的是doGetBean方法。

```java
public Object getBean(String name) throws BeansException {
    return this.doGetBean(name, (Class)null, (Object[])null, false);
}

protected <T> T doGetBean(String name, Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
    // 1.转换beanName
    final String beanName = this.transformedBeanName(name);
    Object sharedInstance = this.getSingleton(beanName);
    Object bean;
    if (sharedInstance != null && args == null) {
        if (this.logger.isDebugEnabled()) {
            if (this.isSingletonCurrentlyInCreation(beanName)) {
                this.logger.debug("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
            } else {
                this.logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }

        bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
    } else {
        // 5.原型模式的依赖检查
        // 只有在单例模式下才会解决依赖循环的问题
        if (this.isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 6.检测ParentBeanFactory
        // 主要是检测如果当前加载的xml文件中不包含beanName所对应的配置，就只能到ParentBeanFactory去尝试了，然后递归调用getBean方法
        BeanFactory parentBeanFactory = this.getParentBeanFactory();
        if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
            String nameToLookup = this.originalBeanName(name);
            if (args != null) {
                return parentBeanFactory.getBean(nameToLookup, args);
            }

            return parentBeanFactory.getBean(nameToLookup, requiredType);
        }

        if (!typeCheckOnly) {
            this.markBeanAsCreated(beanName);
        }

        try {
            // 7.将存储XML配置文件的GenericBeanDefinition转化为RootBeanDefinition
            // 因为从XML配置文件中读取到的bean信息是存储在GenericBeanDefinition中的，但是所有的bean后续处理都是针对于RootBeanDefinition的，所以需要转换，转换的同时如果父类bean不为空的话，则一并合并父类的属性。
            final RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
            this.checkMergedBeanDefinition(mbd, beanName, args);
            // 8.寻找依赖
            // 因为bean在初始化的过程中可能会用到某些属性，而某些属性是动态配置的，并且配置成依赖其他的Bean，这个时候就要先加载依赖的bean
            String[] dependsOn = mbd.getDependsOn();
            String[] var11;
            if (dependsOn != null) {
                var11 = dependsOn;
                int var12 = dependsOn.length;

                for(int var13 = 0; var13 < var12; ++var13) {
                    String dep = var11[var13];
                    if (this.isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }

                    this.registerDependentBean(dep, beanName);
                    this.getBean(dep);
                }
            }

            if (mbd.isSingleton()) {
                     // 2.调用getSingleton方法
                sharedInstance = this.getSingleton(beanName, new ObjectFactory<Object>() {
                    public Object getObject() throws BeansException {
                        try {
                            // 4.调用createBean()方法准备创建bean
                            return AbstractBeanFactory.this.createBean(beanName, mbd, args);
                        } catch (BeansException var2) {
                            AbstractBeanFactory.this.destroySingleton(beanName);
                            throw var2;
                        }
                    }
                });
                 // 3.调用getObjectBeanInstance方法
                bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            } else if (mbd.isPrototype()) {
                var11 = null;

                Object prototypeInstance;
                try {
                    this.beforePrototypeCreation(beanName);
                    prototypeInstance = this.createBean(beanName, mbd, args);
                } finally {
                    this.afterPrototypeCreation(beanName);
                }

                bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            } else {
                // 9.针对不同的scope进行bean的创建
                // Spring会根据不同的配置进行不同的初始化策略
                String scopeName = mbd.getScope();
                Scope scope = (Scope)this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }

                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        public Object getObject() throws BeansException {
                            AbstractBeanFactory.this.beforePrototypeCreation(beanName);

                            Object var1;
                            try {
                                var1 = AbstractBeanFactory.this.createBean(beanName, mbd, args);
                            } finally {
                                AbstractBeanFactory.this.afterPrototypeCreation(beanName);
                            }

                            return var1;
                        }
                    });
                    bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                } catch (IllegalStateException var21) {
                    throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var21);
                }
            }
        } catch (BeansException var23) {
            this.cleanupAfterBeanCreationFailure(beanName);
            throw var23;
        }
    }

    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        try {
            // 10.类型转换
            // 返回的bean可能是个String，但是需要的类型可能是Integer，所以需要转换最后返回我们需要的bean
            return this.getTypeConverter().convertIfNecessary(bean, requiredType);
        } catch (TypeMismatchException var22) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var22);
            }

            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    } else {
        return bean;
    }
}
```

## 1.转换beanName

主要是去除FactoryBean的修饰符，也就是如果name = &aa,首先去除&使得name=aa，还有alias指定的别名。如别名A指向名称为B的bean，则返回B。

```java
protected String transformedBeanName(String name) {
    // 1.在这里调用一个工具类，用来将&去掉，然后再调用规范名称的方法
    return this.canonicalName(BeanFactoryUtils.transformedBeanName(name));
}

// 2.这个方法是为了去除前面的&
  public static String transformedBeanName(String name) {
        Assert.notNull(name, "'name' must not be null");

        String beanName;
        for(beanName = name; beanName.startsWith("&"); beanName = beanName.substring("&".length())) {
        }

        return beanName;
    }

// 3.规范名称
 public String canonicalName(String name) {
        String canonicalName = name;

        String resolvedName;
        do {
            // 从别名表中获取别名指向的bean，一直寻找
            resolvedName = (String)this.aliasMap.get(canonicalName);
            if (resolvedName != null) {
                canonicalName = resolvedName;
            }
        } while(resolvedName != null);

        return canonicalName;
    }
```

## 2.调用getSingleton方法

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "'beanName' must not be null");
    synchronized(this.singletonObjects) {
        // 1.检查SingletonObjects缓存中是否加载过
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName, "Singleton bean creation not allowed while singletons of this factory are in destruction (Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }

            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
			
            // 2.如果没有加载，在加载单例前就记录beanName当前的状态(调用beforeSingletonCreation方法记录)
            this.beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = this.suppressedExceptions == null;
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet();
            }

            try {
                // 3.调用参数传入的ObjectFactory的getObject方法实例化bean
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            } catch (IllegalStateException var16) {
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw var16;
                }
            } catch (BeanCreationException var17) {
                BeanCreationException ex = var17;
                if (recordSuppressedExceptions) {
                    Iterator var8 = this.suppressedExceptions.iterator();

                    while(var8.hasNext()) {
                        Exception suppressedException = (Exception)var8.next();
                        ex.addRelatedCause(suppressedException);
                    }
                }

                throw ex;
            } finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }

                // 4.调用加载单例后的处理方法(afterSingletonCreation())
                this.afterSingletonCreation(beanName);
            }

            if (newSingleton) {
                // 5.将结果记录在SingletonObjects缓存中并删除bean过程所记录的所有辅助状态
                this.addSingleton(beanName, singletonObject);
            }
        }
		// 6.返回处理结果
        return singletonObject != NULL_OBJECT ? singletonObject : null;
    }
}
```
### 2.5.addSingleton

```java
protected void addSingleton(String beanName, Object singletonObject) {
        synchronized(this.singletonObjects) {
			// 1.将结果记录在SingletonObjects缓存中
            this.singletonObjects.put(beanName, singletonObject != null ? singletonObject : NULL_OBJECT);
            // 2.删除bean过程所记录的所有辅助状态
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
```

## 3.调用getObjectForBeanInstance方法

如果从缓存中得到bean的初始状态，则需要对bean进行实例化，缓存中只是记录最原始的Bean状态，并不是我们最终要的bean，比如需要对FactoryBean进行处理，那么从缓存中得到的是工厂Bean的初始状态，但是真正要的是getObject返回的对象。

```java
protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {
    if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
        throw new BeanIsNotAFactoryException(this.transformedBeanName(name), beanInstance.getClass());
    } else if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) { // 1.对FactoryBean正确性的验证
        // 3.对bean进行转换
        Object object = null;
        if (mbd == null) {
            object = this.getCachedObjectForFactoryBean(beanName);
        }

        if (object == null) {
            FactoryBean<?> factory = (FactoryBean)beanInstance;
            if (mbd == null && this.containsBeanDefinition(beanName)) {
                mbd = this.getMergedLocalBeanDefinition(beanName);
            }

            boolean synthetic = mbd != null && mbd.isSynthetic();
            // 4.将从Factory中解析bean的工作委托给getObjectFromFactoryBean
            object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
        }

        return object;
    } else { // 2.对非FactoryBean不做任何处理
        return beanInstance;
    }
}
```
### 3.4.getObjectFromFactoryBean

```java
  protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        if (factory.isSingleton() && this.containsSingleton(beanName)) {
            synchronized(this.getSingletonMutex()) {
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    object = this.doGetObjectFromFactoryBean(factory, beanName);
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    } else {
                        if (object != null && shouldPostProcess) {
                            try {
                                object = this.postProcessObjectFromFactoryBean(object, beanName);
                            } catch (Throwable var9) {
                                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var9);
                            }
                        }

                        this.factoryBeanObjectCache.put(beanName, object != null ? object : NULL_OBJECT);
                    }
                }

                return object != NULL_OBJECT ? object : null;
            }
        } else {
            // getObjectFromFactoryBean调用doGetObjectFromFactoryBean
            Object object = this.doGetObjectFromFactoryBean(factory, beanName);
            if (object != null && shouldPostProcess) {
                try {
                    object = this.postProcessObjectFromFactoryBean(object, beanName);
                } catch (Throwable var11) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", var11);
                }
            }

            return object;
        }
    }
```
#### 3.4.1.doGetObjectFromFactoryBean

```java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, String beanName) throws BeanCreationException {
        Object object;
        try {
            if (System.getSecurityManager() != null) { // 2.调用ObjectFactory的后处理器
                AccessControlContext acc = this.getAccessControlContext();

                try {
                    object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                        public Object run() throws Exception {
                            return factory.getObject();
                        }
                    }, acc);
                } catch (PrivilegedActionException var6) {
                    throw var6.getException();
                }
            } else { //1.如果声明的bean是FactoryBean类型，则直接调用getObject返回对象
                object = factory.getObject();
            }
        } catch (FactoryBeanNotInitializedException var7) {
            throw new BeanCurrentlyInCreationException(beanName, var7.toString());
        } catch (Throwable var8) {
            throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", var8);
        }

        if (object == null && this.isSingletonCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName, "FactoryBean which is currently in creation returned null from getObject");
        } else {
            return object;
        }
    }
```

## 4.调用createBean()方法准备创建bean

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Creating instance of bean '" + beanName + "'");
    }

    RootBeanDefinition mbdToUse = mbd;
    // 1.根据设置的class属性或者根据className来解析class
    Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    try {
        // 2.调用prepareMethodOverrides方法对override属性进行标记及验证(spring的配置中是存在lookup-method和replace-method的,这两个配置的加载统一放到BeanDefinition中的methodOverrides属性里，createBean主要是针对这两个配置的)
        mbdToUse.prepareMethodOverrides();
    } catch (BeanDefinitionValidationException var7) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(), beanName, "Validation of method overrides failed", var7);
    }

    Object beanInstance;
    try {
        // 3.应用初始化前的后处理器(调用resolveBeforeInstantiation方法)，解析指定bean是否存在初始化前的短路操作
        beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
        // 调用resolveBeforeInstantiation后返回的bean如果不为Null，直接返回，不用再处理后续的doCreateBean方法了。
        if (beanInstance != null) {
            return beanInstance;
        }
    } catch (Throwable var8) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var8);
    }

    // 4.调用doCreateBean创建Bean
    beanInstance = this.doCreateBean(beanName, mbdToUse, args);
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Finished creating instance of bean '" + beanName + "'");
    }

    return beanInstance;
}
```
### 4.2.prepareMethodOverrides
```java
// 在bean实例化的时候如果检测到存在methodOverrides属性，会动态的为当前Bean生成代理对象并使用对应的拦截器为bean做增强处理
// 如果当前类存在若干个重载方法，那么在函数调用的时候以及增强的时候需要根据参数类型进行匹配，来确定最终当前调用的到底是哪个方法
public void prepareMethodOverrides() throws BeanDefinitionValidationException {
        MethodOverrides methodOverrides = this.getMethodOverrides();
        if (!methodOverrides.isEmpty()) {
            Set<MethodOverride> overrides = methodOverrides.getOverrides();
            synchronized(overrides) {
                Iterator var4 = overrides.iterator();

                // 1.遍历获取到的methodOverrides
                while(var4.hasNext()) { 
                    MethodOverride mo = (MethodOverride)var4.next();
                    // 2.对每一个methodOverride调用prePareMethodOverride方法
                    this.prepareMethodOverride(mo);
                }
            }
        }

    }
```
#### 4.2.2.prepareMethodOverride
```java
protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
    	// 1.获取对应类中对应方法名的个数
        int count = ClassUtils.getMethodCountForName(this.getBeanClass(), mo.getMethodName());
    	// 2.如果为0，就抛异常，为方法的存在进行验证
        if (count == 0) {
            throw new BeanDefinitionValidationException("Invalid method override: no method with name '" + mo.getMethodName() + "' on class [" + this.getBeanClassName() + "]");
        } else { // 3.如果当前类只有一个，那么就设置该方法没有被重载，这样在后续调用的时候就可以直接找到对应的方法
            if (count == 1) {
                mo.setOverloaded(false);
            }

        }
    }
```
### 4.3.resolveBeforeInstantiation
```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = this.determineTargetType(beanName, mbd);
                if (targetType != null) {
                    // resolveBeforeInstantiation先调用applyBeanPostProcessorsBeforeInstantiation，如果得到的bean不为Null就调用applyBeanPostProcessorsAfterInstantiation,在方法最后return bean。
                    // applyBeanPostProcessorsBeforeInstantiation：给子类修改BeanDefination的机会，完成这个方法后，bean可能已经不是我们认为的Bean了，获取成为了一个经过代理的代理bean。
                    bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        // applyBeanPostProcessorsAfterInstantiation:因为如果调用的resolveBeforeInstantiation方法返回的bean不为null，那么便不会再次经历普通Bean的创建过程，所以只能在这里应用后处理器的applyBeanPostProcessorsAfterInstantiation方法
                        bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }

            mbd.beforeInstantiationResolved = bean != null;
        }

        return bean;
    }
```
### 4.4.doCreateBean
```java
// 调用doCreateBean创建Bean
 protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        BeanWrapper instanceWrapper = null;
     	// 1.如果是单例则清除缓存
        if (mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if (instanceWrapper == null) {
            // 2.调用createBeanInstance方法实例化bean，将BeanDefinition转换为BeanWrapper
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        final Object bean = instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null;
        Class<?> beanType = instanceWrapper != null ? instanceWrapper.getWrappedClass() : null;
        mbd.resolvedTargetType = beanType;
        synchronized(mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                    // 3.调用MergedBeanDefinitionPostProcessor的applyMergedDefinitionPostProcessors方法
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                }

                mbd.postProcessed = true;
            }
        }

        boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }

            // 4.循环依赖处理，为了避免后期循环依赖，在真正bean初始化完成之前将创建实例的ObjectFactory放入工厂
            this.addSingletonFactory(beanName, new ObjectFactory<Object>() {
                public Object getObject() throws BeansException {
                    return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
                }
            });
        }

        Object exposedObject = bean;

        try {
            // 5.调用populateBean方法进行属性填充，将所有的属性填充到bean的实例中，如果bean不为空，就初始化
            this.populateBean(beanName, mbd, instanceWrapper);
            // 6.如果bean不为null，调用初始化方法initialzeBean方法
            if (exposedObject != null) {
                exposedObject = this.initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
        }

        if (earlySingletonExposure) {
            Object earlySingletonReference = this.getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    // 7.循环依赖检查，如果不是单例就抛异常
                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }

        try {
            // 8.注册DisposableBean，如果配置了destroy-method，这里就需要注册以便于销毁时调用注册用户自定义的销毁方法
            this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
            // 9.完成创建并返回
            return exposedObject;
        } catch (BeanDefinitionValidationException var16) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
        }
    }
```
#### 4.4.2.createBeanInstance
```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
        Class<?> beanClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
        } else if (mbd.getFactoryMethodName() != null) { // 1.如果在RootBeanDefination中存在factoryMethodName属性，或者在配置文件中配置了factory-method，那么spring会调用一个方法(instantiateUsingFactoryMethod()方法)，根据RootBeanDefinition中的配置生成Bean
            return this.instantiateUsingFactoryMethod(beanName, mbd, args);
        } else {
            boolean resolved = false;
            boolean autowireNecessary = false;
            if (args == null) {
                synchronized(mbd.constructorArgumentLock) {
                    if (mbd.resolvedConstructorOrFactoryMethod != null) {
                        resolved = true;
                        autowireNecessary = mbd.constructorArgumentsResolved;
                    }
                }
            }
			
            // 2.(调用autowiredConstructor方法自动注入，或者InstantiateBean方法使用默认构造函数构造)解析构造函数并进行构造函数的实例化，spring中可能有多个构造函数，spring根据参数以及类型去判断最终会使用哪个构造函数进行实例化，如果已经解析过则不需要重复解析构造函数，而是直接从RootBeanDefination中的属性resolvedConstructorOrFactoryMethod缓存的值去取，否则需要再次解析，将解析的结果放入RootBeanDefination中的属性resolvedConstructorOrFactoryMethod缓存中。
            if (resolved) {
                return autowireNecessary ? this.autowireConstructor(beanName, mbd, (Constructor[])null, (Object[])null) : this.instantiateBean(beanName, mbd);
            } else {
                Constructor<?>[] ctors = this.determineConstructorsFromBeanPostProcessors(beanClass, beanName);
                return ctors == null && mbd.getResolvedAutowireMode() != 3 && !mbd.hasConstructorArgumentValues() && ObjectUtils.isEmpty(args) ? this.instantiateBean(beanName, mbd) : this.autowireConstructor(beanName, mbd, ctors, args);
            }
        }
    }
```
##### 4.4.2.1.autowiredConstructor
```java
// autowiredConstructor方法
// 1.构造函数参数的确定
// A.根据传入的explicitArgs参数判断(在获取Bean的时候用户可指定bean的名称以及bean所对应类的构造函数或者工厂方法的方法参数，所以，如果用户传入了explictArgs参数不为空，就可以确定构造函数参数就是这个参数)
// B.如果构造函数参数已经记录在缓存中，那么就可以直接拿来使用，但是缓存中缓存的可能参数的最终类型也可能是参数的初始类型，例如构造函数参数 要求int类型，缓存中是String类型，那么即使在缓存中得到了参数，也需要经过类型转换器的过滤以保证参数类型与对应的构造函数的参数类型完全对应
 // C.如果不能根据explicitArgs确定构造函数的参数也无法在缓存中得到相关信息，就分析从配置文件中配置的构造函数，因为spring中配置文件的信息通过转换都会通过BeanDefinition实例承载，也就是方法中mbd(rootBeanDefination)中包含，就可以通过mdb.getConstructorArgumentValues方法来获取配置的构造函数信息，通过这些配置信息就可以获取对应的参数值信息
// 2.构造函数的确定
// A.先对构造函数按照public构造函数优先参数数量降序，非Public构造函数参数数量降序
// B.在配置文件中可以通过参数位置索引以及指定参数名称进行设定参数值，通过spring提供的工具类ParameterNameDiscover来获取参数名称，这样就可以确定构造函数，参数类型，参数名称，参数值这些信息了
// C.根据确定的构造函数转换对应的参数类型
// D.构造函数不确定性的验证，验证一些比如参数为父子关系的
// E.根据实例化策略((调用instantiate方法))以及得到的构造函数以及构造函数的参数实例化bean
public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd, Constructor<?>[] chosenCtors, Object[] explicitArgs) {
        BeanWrapperImpl bw = new BeanWrapperImpl();
        this.beanFactory.initBeanWrapper(bw);
        final Constructor<?> constructorToUse = null;
        ConstructorResolver.ArgumentsHolder argsHolderToUse = null;
        final Object[] argsToUse = null;
    
        if (explicitArgs != null) { 
            argsToUse = explicitArgs;
        } else {
            Object[] argsToResolve = null;
            synchronized(mbd.constructorArgumentLock) {
                constructorToUse = (Constructor)mbd.resolvedConstructorOrFactoryMethod;
                if (constructorToUse != null && mbd.constructorArgumentsResolved) {

                    argsToUse = mbd.resolvedConstructorArguments;
                    if (argsToUse == null) {
                        argsToResolve = mbd.preparedConstructorArguments;
                    }
                }
            }

            if (argsToResolve != null) {
                argsToUse = this.resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
            }
        }

        if (constructorToUse == null) {
            boolean autowiring = chosenCtors != null || mbd.getResolvedAutowireMode() == 3;
            ConstructorArgumentValues resolvedValues = null;
            int minNrOfArgs;
            if (explicitArgs != null) {
                minNrOfArgs = explicitArgs.length;
            } else {
                ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
                resolvedValues = new ConstructorArgumentValues();
                minNrOfArgs = this.resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
            }

            Constructor<?>[] candidates = chosenCtors; // candidates候选人
            if (chosenCtors == null) {
                Class beanClass = mbd.getBeanClass();

                try {
                    candidates = mbd.isNonPublicAccessAllowed() ? beanClass.getDeclaredConstructors() : beanClass.getConstructors();
                } catch (Throwable var25) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Resolution of declared constructors on bean Class [" + beanClass.getName() + "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", var25);
                }
            }

            AutowireUtils.sortConstructors(candidates);
            int minTypeDiffWeight = 2147483647;
            Set<Constructor<?>> ambiguousConstructors = null;
            LinkedList<UnsatisfiedDependencyException> causes = null;
            Constructor[] var16 = candidates;
            int var17 = candidates.length;

            for(int var18 = 0; var18 < var17; ++var18) {
                Constructor<?> candidate = var16[var18];
                Class<?>[] paramTypes = candidate.getParameterTypes();
                if (constructorToUse != null && argsToUse.length > paramTypes.length) {
                    break;
                }

                if (paramTypes.length >= minNrOfArgs) {
                    ConstructorResolver.ArgumentsHolder argsHolder;
                    if (resolvedValues != null) {
                        try {
                            String[] paramNames = ConstructorResolver.ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
                            if (paramNames == null) {
                                ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                                if (pnd != null) {
                                    paramNames = pnd.getParameterNames(candidate);
                                }
                            }

                            argsHolder = this.createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames, this.getUserDeclaredConstructor(candidate), autowiring);
                        } catch (UnsatisfiedDependencyException var26) {
                            if (this.beanFactory.logger.isTraceEnabled()) {
                                this.beanFactory.logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + var26);
                            }

                            if (causes == null) {
                                causes = new LinkedList();
                            }

                            causes.add(var26);
                            continue;
                        }
                    } else {
                        if (paramTypes.length != explicitArgs.length) {
                            continue;
                        }

                        argsHolder = new ConstructorResolver.ArgumentsHolder(explicitArgs);
                    }

                    int typeDiffWeight = mbd.isLenientConstructorResolution() ? argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes);
                    if (typeDiffWeight < minTypeDiffWeight) {
                        constructorToUse = candidate;
                        argsHolderToUse = argsHolder;
                        argsToUse = argsHolder.arguments;
                        minTypeDiffWeight = typeDiffWeight;
                        ambiguousConstructors = null;
                    } else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
                        if (ambiguousConstructors == null) {
                            ambiguousConstructors = new LinkedHashSet();
                            ambiguousConstructors.add(constructorToUse);
                        }

                        ambiguousConstructors.add(candidate);
                    }
                }
            }

            if (constructorToUse == null) {
                if (causes == null) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Could not resolve matching constructor (hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
                }

                UnsatisfiedDependencyException ex = (UnsatisfiedDependencyException)causes.removeLast();
                Iterator var34 = causes.iterator();

                while(var34.hasNext()) {
                    Exception cause = (Exception)var34.next();
                    this.beanFactory.onSuppressedException(cause);
                }

                throw ex;
            }

            if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Ambiguous constructor matches found in bean '" + beanName + "' (hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " + ambiguousConstructors);
            }

            if (explicitArgs == null) {
                argsHolderToUse.storeCache(mbd, constructorToUse);
            }
        }

        try {
            Object beanInstance;
            if (System.getSecurityManager() != null) {
                beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    public Object run() {
                        return ConstructorResolver.this.beanFactory.getInstantiationStrategy().instantiate(mbd, beanName, ConstructorResolver.this.beanFactory, constructorToUse, argsToUse);
                    }
                }, this.beanFactory.getAccessControlContext());
            } else {
                beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
            }

            bw.setBeanInstance(beanInstance);
            return bw;
        } catch (Throwable var24) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Bean instantiation via constructor failed", var24);
        }
    }
```
##### 4.4.2.2.instantiateBean
```java
//instantiateBean方法，调用默认的构造方法构造bean
// 该方法直接调用实例化策略进行实例化(调用instantiate方法)
// 实例化策略？
// A.首先判断如果beanDefinition.getMethodOverrides()为空也就是用户没有使用replace或者lookup的配置方法
// B.如果没有呢，就直接使用反射的方式
// C.如果使用了这两个配置方法，就要使用动态代理的方式将包含两个特性所对应的逻辑的拦截增强器设置进去
  protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
        try {
            Object beanInstance;
            if (System.getSecurityManager() != null) {
                beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    public Object run() {
                        return AbstractAutowireCapableBeanFactory.this.getInstantiationStrategy().instantiate(mbd, beanName, AbstractAutowireCapableBeanFactory.this);
                    }
                }, this.getAccessControlContext());
            } else {
                beanInstance = this.getInstantiationStrategy().instantiate(mbd, beanName, this);
            }

            BeanWrapper bw = new BeanWrapperImpl(beanInstance);
            this.initBeanWrapper(bw);
            return bw;
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", var6);
        }
    }

```
#### 4.4.4.解决循环依赖

 解决循环依赖问题，比如有两个bean，A依赖B，B依赖A，创建A的时候，创建之前A的ObjectFactory会存入缓存中，而在对A的属性进行填充时，也就是调用populateBean方法又会对B进行创建，因为B有A，因此B在使用populateBean方法进行属性填充的时候，又会初始化A，但是在该方法中，并不是直接去实例化A，而是先从缓存中是否有创建好的bean或者已经建好了ObjectFactory，因此ObjectFactory已经建好了，所以直接调用ObjectFactory去创建A，B初始化完后，对A的属性进行填充。

在spring中创建bean的原则是不等bean创建完成就会创建bean的对象工厂(ObjectFactory)提早曝光加入缓存中，避免单例下的循环依赖。

1.首先尝试从(单例对象)SingletonObjects里面获取单例

2.如果获取不到再从earlySingletonObject中获取

3.如果还获取不到就从singletonFactories中获取beanName对应的ObjectFactory

4.然后调用ObjectFactory的getObject来创建bean，并放到earlySingletonObject里面去，然后从singletonFactories里面remove掉这个ObjectFactory。



SingletonObjects：保存beanName和创建bean实例之间的关系

earlySingletonObject：保存beanName和创建bean实例之间的关系，和SingletonObjects不同的是，当一个单例Bean被放到这里面后，那么当bean还在创建过程中就可以通过getBean方法获取到了，目的是用来检测循环引用。

singletonFactories：用于保存beanName和创建Bean的工厂之间的关系



循环依赖：

构造器循环依赖是无法解决的，只能抛出异常

Setter循环依赖，只能解决单例模式下的循环依赖

Prototype模式的依赖：因为spring容器不进行缓存prototype作用域的bean，因此无法提前暴露一个创建中的bean，所以模式无法完成依赖注入。

```java
boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }

            this.addSingletonFactory(beanName, new ObjectFactory<Object>() {
                public Object getObject() throws BeansException {
                    return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
                }
            });
        }

        Object exposedObject = bean;

        try {
            this.populateBean(beanName, mbd, instanceWrapper);
            if (exposedObject != null) {
                exposedObject = this.initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
        }

        if (earlySingletonExposure) {
            Object earlySingletonReference = this.getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }



 protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            synchronized(this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }

        return singletonObject != NULL_OBJECT ? singletonObject : null;
    }
```





#### 4.4.5.populateBean

```java
 protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
        PropertyValues pvs = mbd.getPropertyValues();
        if (bw == null) {
            if (!((PropertyValues)pvs).isEmpty()) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            }
        } else {
            boolean continueWithPropertyPopulation = true;
            // 1.先判断是否继续填充属性，如果后处理器发出停止命令则终止后面的执行
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Iterator var6 = this.getBeanPostProcessors().iterator();

                while(var6.hasNext()) {
                    BeanPostProcessor bp = (BeanPostProcessor)var6.next();
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                            continueWithPropertyPopulation = false;
                            break;
                        }
                    }
                }
            }

            if (continueWithPropertyPopulation) {
                // 2.根据注入类型(byName/byType)，提取依赖的bean，并统一存入PropertyValues中
                if (mbd.getResolvedAutowireMode() == 1 || mbd.getResolvedAutowireMode() == 2) {
                    MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
                    if (mbd.getResolvedAutowireMode() == 1) {
                        this.autowireByName(beanName, mbd, bw, newPvs);
                    }

                    if (mbd.getResolvedAutowireMode() == 2) {
                        this.autowireByType(beanName, mbd, bw, newPvs);
                    }

                    pvs = newPvs;
                }

                boolean hasInstAwareBpps = this.hasInstantiationAwareBeanPostProcessors();
                boolean needsDepCheck = mbd.getDependencyCheck() != 0;
                // 3.对属性获取完毕填充前对属性再次验证
                if (hasInstAwareBpps || needsDepCheck) {
                    PropertyDescriptor[] filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    if (hasInstAwareBpps) {
                        Iterator var9 = this.getBeanPostProcessors().iterator();

                        while(var9.hasNext()) {
                            BeanPostProcessor bp = (BeanPostProcessor)var9.next();
                            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                                pvs = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds, bw.getWrappedInstance(), beanName);
                                if (pvs == null) {
                                    return;
                                }
                            }
                        }
                    }

                    if (needsDepCheck) {
                        this.checkDependencies(beanName, mbd, filteredPds, (PropertyValues)pvs);
                    }
                }

                // 4.将所有的PropertyValues中的属性填充到BeanWrapper中
                this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
            }
        }
    }
```

