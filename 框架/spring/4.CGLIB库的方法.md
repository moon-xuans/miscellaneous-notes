# Cglib动态代理实现方式

## 1.Cglib动态代理实现方式

我们先来看个例子，看一下Cglib如何实现动态代理的。

首先定义一个服务类，有两个方法并且其中一个方法用final来修饰。

```java
public class PersonService {
    public PersonService() {
        System.out.println("PersonService构造");
    }

    // 该方法不能被子类覆盖
    final public Person getPerson(String code) {
        System.out.println("PersonService:getPerson>>" + code);
        return null;
    }

    public void setPerson() {
        System.out.println("PersonService:setPerson");
    }
}
```

Cglib是无法代理final修饰的方法的，具体原因我们一会通过源码来分析。

然后，定义一个自定义MethodInterceptor。

```java
public class CglibProxyInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("执行前...");
        Object object = methodProxy.invokeSuper(o, objects);
        System.out.println("执行后...");
        return object;
    }
}
```

我们看一下interceptor方法入参，o:cglib生成的代理对象，method:被代理对象，objects:方法入参,methodProxy:代理方法。

最后，我们写个例子调用以下，并将Cglib生成的代理类class文件输出磁盘方便我们反编译查看源码。

```java
public class Test {
    public static void main(String[] args) {
        // 代理类文件存入本地磁盘
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\test");
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(PersonService.class);
        enhancer.setCallback(new CglibProxyInterceptor());
        PersonService proxy = (PersonService) enhancer.create();
        proxy.setPerson();
        proxy.getPerson("1");
    }
}
```

我们执行一下会发现getPerson因为加final修饰并没有被代理，下面我们通过源码分析一下。

![QQ图片20220207100525](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/QQ%E5%9B%BE%E7%89%8720220207100525.png)

## 2.生成代理类

执行Test测试类可以得到Cglib生成的class文件，一共有三个class文件我们反编译以后逐个说一下他们的作用。

![image-20220207100839322](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220207100839322.png)

PersonService$$EnhancerByCGLIB$$b63c621就是cglib生成的代理类，它继承了PersonService类。

```java
public class PersonService$$EnhancerByCGLIB$$b63c621 extends PersonService
    implements Factory
{

    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback CGLIB$STATIC_CALLBACKS[];
    private MethodInterceptor CGLIB$CALLBACK_0;  // 拦截器
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$setPerson$0$Method;  // 被代理方法
    private static final MethodProxy CGLIB$setPerson$0$Proxy; // 代理方法
    private static final Object CGLIB$emptyArgs[];
    private static final Method CGLIB$equals$1$Method;
    private static final MethodProxy CGLIB$equals$1$Proxy;
    private static final Method CGLIB$toString$2$Method;
    private static final MethodProxy CGLIB$toString$2$Proxy;
    private static final Method CGLIB$hashCode$3$Method;
    private static final MethodProxy CGLIB$hashCode$3$Proxy;
    private static final Method CGLIB$clone$4$Method;
    private static final MethodProxy CGLIB$clone$4$Proxy;
    
    static void CGLIB$STATICHOOK1()
    {
        Method amethod[];
        Method amethod1[];
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class class1 = Class.forName("com.axuan.aop.cglib.PersonService$$EnhancerByCGLIB$$b63c621"); // 代理类
        Class class2; // 被代理类PersonService
        amethod = ReflectUtils.findMethods(new String[] {
            "equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"
        }, (class2 = Class.forName("java.lang.Object")).getDeclaredMethods());
        Method[] _tmp = amethod;
        CGLIB$equals$1$Method = amethod[0];
        CGLIB$equals$1$Proxy = MethodProxy.create(class2, class1, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$1");
        CGLIB$toString$2$Method = amethod[1];
        CGLIB$toString$2$Proxy = MethodProxy.create(class2, class1, "()Ljava/lang/String;", "toString", "CGLIB$toString$2");
        CGLIB$hashCode$3$Method = amethod[2];
        CGLIB$hashCode$3$Proxy = MethodProxy.create(class2, class1, "()I", "hashCode", "CGLIB$hashCode$3");
        CGLIB$clone$4$Method = amethod[3];
        CGLIB$clone$4$Proxy = MethodProxy.create(class2, class1, "()Ljava/lang/Object;", "clone", "CGLIB$clone$4");
        amethod1 = ReflectUtils.findMethods(new String[] {
            "setPerson", "()V"
        }, (class2 = Class.forName("com.axuan.aop.cglib.PersonService")).getDeclaredMethods());
        Method[] _tmp1 = amethod1;
        CGLIB$setPerson$0$Method = amethod1[0];
        CGLIB$setPerson$0$Proxy = MethodProxy.create(class2, class1, "()V", "setPerson", "CGLIB$setPerson$0");
    }
```

我们通过代理类的源码可以看到，代理类会获得所有在父类继承来的方法，并且会有MethodProxy与之对应，比如Method CGLIB$setPerson$0$Method、MethodProxy CGLIB$setPerson$0$Proxy;

## 3.方法的调用

```java
    // 代理方法(methodProxy.invokeSuper会使用)
    final void CGLIB$setPerson$0()
        {
            super.setPerson();
        }

	// 被代理方法(methodProxy.invoke会调用，这也就是为什么拦截器中调用methodProxy.invoke会死循环，一直在调用拦截器)
    public final void setPerson()
    {
        CGLIB$CALLBACK_0;
        if(CGLIB$CALLBACK_0 != null) goto _L2; else goto _L1
_L1:
        JVM INSTR pop ;
        CGLIB$BIND_CALLBACKS(this);
        CGLIB$CALLBACK_0;
_L2:
        JVM INSTR dup ;
        JVM INSTR ifnull 37;
           goto _L3 _L4
_L3:
        break MISSING_BLOCK_LABEL_21;
_L4:
        break MISSING_BLOCK_LABEL_37;
        this;
        CGLIB$setPerson$0$Method;
        CGLIB$emptyArgs;
        CGLIB$setPerson$0$Proxy;
        // 调用拦截器
        intercept();
        return;
        super.setPerson();
        return;
    }
```

调用过程：代理对象调用this.setPerson方法->调用拦截器->methodProxy.invokeSuper->CGLIB$setPerson$0->被代理对象setPerson方法

## 4.MethodProxy

拦截器MethodInterceptor中就是由MethodProxy的invokeSuper方法调用代理方法的，MethodProxy非常关键，我们分析一下它具体做了什么。

- 创建MethodProxy

```java
public class MethodProxy {
    private Signature sig1;
    private Signature sig2;
    private MethodProxy.CreateInfo createInfo;
    private final Object initLock = new Object();
    private volatile MethodProxy.FastClassInfo fastClassInfo;
    //c1:被代理对象Class
    //c2:代理对象Class
    //desc：入参类型
    //name1:被代理方法名
    //name2:代理方法名
    public static MethodProxy create(Class c1, Class c2, String desc, String name1, String name2) {
        MethodProxy proxy = new MethodProxy();
        proxy.sig1 = new Signature(name1, desc);//被代理方法签名
        proxy.sig2 = new Signature(name2, desc);//代理方法签名
        proxy.createInfo = new MethodProxy.CreateInfo(c1, c2);
        return proxy;
    }
private static class CreateInfo {
    Class c1;
    Class c2;
    NamingPolicy namingPolicy;
    GeneratorStrategy strategy;
    boolean attemptLoad;

    public CreateInfo(Class c1, Class c2) {
        this.c1 = c1;
        this.c2 = c2;
        AbstractClassGenerator fromEnhancer = AbstractClassGenerator.getCurrent();
        if(fromEnhancer != null) {
            this.namingPolicy = fromEnhancer.getNamingPolicy();
            this.strategy = fromEnhancer.getStrategy();
            this.attemptLoad = fromEnhancer.getAttemptLoad();
        }

    }
}
```

- invokeSuper调用

```java
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
        try {
            this.init();
            MethodProxy.FastClassInfo fci = this.fastClassInfo;
            return fci.f2.invoke(fci.i2, obj, args);
        } catch (InvocationTargetException var4) {
            throw var4.getTargetException();
        }
    }
private static class FastClassInfo {
    FastClass f1;//被代理类FastClass
    FastClass f2;//代理类FastClass
    int i1; //被代理类的方法签名(index)
    int i2;//代理类的方法签名

    private FastClassInfo() {
    }
}
```

上面代码调用过程就是获取到代理类对应的FastClass，并执行了代理方法。还记得之前生成三个class文件吗？PersonService$$EnhancerByCGLIB$$b63c621$$FastClassByCGLIB$$4c89c2c2.class就是代理类的FastClass，PersonService$$FastClassByCGLIB$$5ea72f37.class就是被代理类的FastClass。

## 5.FastClass机制

Cglib动态代理执行代理方法效率之所以比JDK的高是因为Cglib采用了FastClass机制，它的原理简单来说就是：为代理类和被代理类各生成一个Class，这个Class会为代理类或被代理类的方法分配一个index(int类型)。
这个index当做一个入参，FastClass就可以直接定位要调用的方法直接进行调用，这样省去了反射调用，所以调用效率比JDK动态代理通过反射调用高。下面我们反编译一个FastClass看看：

```java
// 根据方法签名获取index
public int getIndex(Signature signature)
    {
        String s = signature.toString();
        s;
        s.hashCode();
        JVM INSTR lookupswitch 5: default 110
    //                   -1902447170: 60
    //                   1826985398: 70
    //                   1889565678: 80
    //                   1913648695: 90
    //                   1984935277: 100;
           goto _L1 _L2 _L3 _L4 _L5 _L6
_L2:
        "setPerson()V";
        equals();
        JVM INSTR ifeq 111;
           goto _L7 _L8
_L8:
        break MISSING_BLOCK_LABEL_111;
_L7:
        return 1;
_L3:
        "equals(Ljava/lang/Object;)Z";
        equals();
        JVM INSTR ifeq 111;
           goto _L9 _L10
_L10:
        break MISSING_BLOCK_LABEL_111;
_L9:
        return 2;
_L4:
        "getPerson(Ljava/lang/String;)Lcom/axuan/aop/staticproxy/Person;";
        equals();
        JVM INSTR ifeq 111;
           goto _L11 _L12
_L12:
        break MISSING_BLOCK_LABEL_111;
_L11:
        return 0;
_L5:
        "toString()Ljava/lang/String;";
        equals();
        JVM INSTR ifeq 111;
           goto _L13 _L14
_L14:
        break MISSING_BLOCK_LABEL_111;
_L13:
        return 3;
_L6:
        "hashCode()I";
        equals();
        JVM INSTR ifeq 111;
           goto _L15 _L16
_L16:
        break MISSING_BLOCK_LABEL_111;
_L15:
        return 4;
_L1:
        JVM INSTR pop ;
        return -1;
    }

// 根据index直接定位执行方法
 public Object invoke(int i, Object obj, Object aobj[])
        throws InvocationTargetException
    {
        (PersonService)obj;
        i;
        JVM INSTR tableswitch 0 4: default 86
    //                   0 40
    //                   1 50
    //                   2 55
    //                   3 70
    //                   4 74;
           goto _L1 _L2 _L3 _L4 _L5 _L6
_L2:
        (String)aobj[0];
        getPerson();
        return;
_L3:
        setPerson();
        return null;
_L4:
        aobj[0];
        equals();
        JVM INSTR new #74  <Class Boolean>;
        JVM INSTR dup_x1 ;
        JVM INSTR swap ;
        Boolean();
        return;
_L5:
        toString();
        return;
_L6:
        hashCode();
        JVM INSTR new #81  <Class Integer>;
        JVM INSTR dup_x1 ;
        JVM INSTR swap ;
        Integer();
        return;
        JVM INSTR new #63  <Class InvocationTargetException>;
        JVM INSTR dup_x1 ;
        JVM INSTR swap ;
        InvocationTargetException();
        throw ;
_L1:
        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }
```

FastClass并不是跟代理类一块生成的，而是在第一次执行MethodProxy invoke/invokeSuper时成功的并放在了缓存中。

```java
//MethodProxy invoke/invokeSuper都调用了init()
private void init() {
        if(this.fastClassInfo == null) {
            Object var1 = this.initLock;
            synchronized(this.initLock) {
                if(this.fastClassInfo == null) {
                    MethodProxy.CreateInfo ci = this.createInfo;
                    MethodProxy.FastClassInfo fci = new MethodProxy.FastClassInfo();
                    fci.f1 = helper(ci, ci.c1);//如果缓存中就取出，没有就生成新的FastClass
                    fci.f2 = helper(ci, ci.c2);
                    fci.i1 = fci.f1.getIndex(this.sig1);//获取方法的index
                    fci.i2 = fci.f2.getIndex(this.sig2);
                    this.fastClassInfo = fci;
                    this.createInfo = null;
                }
            }
        }

    }
```

至此，Cglib动态代理的原理我们就基本搞清楚了。
最后我们总结一下JDK动态代理和Gglib动态代理的区别：
1.JDK动态代理是实现了被代理对象的接口，Cglib是继承了被代理对象。
2.JDK和Cglib都是在运行期生成字节码，JDK是直接写Class字节码，Cglib使用ASM框架写Class字节码，Cglib代理实现更复杂，生成代理类比JDK效率低。
3.JDK调用代理方法，是通过反射机制调用，Cglib是通过FastClass机制直接调用方法，Cglib执行效率更高。



转自:https://www.cnblogs.com/monkey0307/p/8328821.html