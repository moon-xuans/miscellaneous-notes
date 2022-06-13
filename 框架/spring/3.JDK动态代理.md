# JDK动态代理
在Spring框架中经典的AOP就是通过动态代理来实现的，Spring分别采取了JDK的动态代理和Cglib动态代理，本文来分析一下JDK如何实现动态代理的。

JDK的动态代理是基于接口实现的，所以我们被代理的对象必须有一个接口。

## 1.实现方法

先看下最核心的一个接口和一个方法
这个是java.lang.reflect包里的InvocationHandler接口：

```java
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```
我们对于被代理的类的操作都会由改接口中的invoke方法实现，其中的参数的含义分别是：
- proxy:被代理的类的实例
- method:调用被代理的类的方法
- args:该方法需要的参数

使用方法首先是需要实现该接口，并且我们可以在invoke方法中调用被代理类的方法并获得返回值，自然也可以在调用该方法的前后去做一些额外的事情，从而实现动态代理。

另外一个很重要的静态方法就是java.lang.reflect包中的Proxy类的newProxyInstance方法：
```java
 public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
```
其中的参数含义如下：
- loader：被代理的类的类加载器
- interfaces:被代理类的接口数组
- invocationHandler:就是刚刚介绍的调用处理器类的对象实例

该方法会返回一个被修改过的类的实例，从而可以自由的调用该实例的方法。

## 2.举个例子

下面是一个实际例子。

```java
public interface UserService {

     void display(String userid);
}

public class UserServiceImpl implements UserService {

    @Override
    public void display(String userId) {
        System.out.println("display:" + userId);
    }
}

public class MyInvocationHandler implements InvocationHandler {

    // 被代理对象，java代理模式下的一个必要因素就是代理对象要能拿到被代理对象的引用
    private Object target;

    public MyInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object object = method.invoke(target, args);
        System.out.println("执行后");
        return object;
    }
}
```
测试类：
```java
public class ReflectTest {

    public static void main(String[] args) {
        // 注意一定要返回接口，不能返回实现类否则会报错
        Person person = (Person) DynamicAgent.agent(Person.class, new Actor());
        person.speak();
    }
}

```
结果：

![image-20220207090342257](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220207090342257.png)

可以看到对于不同的实现类来说，可以用同一个动态代理类来进行处理，实现了“一次编写到处代理”的效果。但是这种方法有个缺点，就是被代理的类一定要是实现了某个接口的，这很大程度限制了本方法的使用场景。

## 3.源码分析

```java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    // 1.校验InvocationHandler不能为空
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    // 2.进行权限校验
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    // 3.关键代码，根据classLoader、interfaces生成代理类字节码，直接生成.class不是生成.java
    Class<?> cl = getProxyClass0(loader, intfs);

    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        // 4.获取代理类的构造器(构造器入参:{InvocationHandler.class})
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        // 5.如果代理类是不可访问的，通过特权将它设置它的构造器为可访问的
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        // 6.通过构造器生成代理对象并返回
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

大体上可以分为三步:

1. 通过classLoader、interfaces获取代理类Class；
2. 通过代理类Class获取入参为{InvocationHandler}的构造器；
3. 通过构造器实例化代理对象。

其中，第一步是最关键的代码。

继续深入。

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    // 1.接口数不能大于65535
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
	
    // 2.如果代理类已存在于缓存中直接获取，如果不存在通过ProxyClassFactory生成并返回。
    return proxyClassCache.get(loader, interfaces);
}
```

这里如果缓存中没有对应的代理类就调用ProxyClassFactory的apply方法生成。

```java
@Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        // 1.对传入的数据进行校验
        for (Class<?> intf : interfaces) {
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
             
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // package to define proxy class in
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        // 2.如果代理类实现的接口是非public的，代理类和它实现的接口必须在一个包下
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";  // PROXY_PACKAGE = "com.sun.proxy"
        }

        long num = nextUniqueNumber.getAndIncrement();
        // 代理类名com.sun.proxy.$Proxy0
        String proxyName = proxyPkg + proxyClassNamePrefix + num; // proxyClassNamePrefix = "$Proxy"

        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            // 3.生成Class并加载到JVM
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

ProxyClassFactory.apply方法总结下来就是这几步:

1. 校验入参Class<?> interfaces,包括classLoader加载的interfaces是否与入参是同一个Object、interfaceClass必须是接口类型、interfaces集合中不能重复(通过IdentityHashMap的特性，key值相等比的是地址，即key1=key2)。
2. 定义代理类的包名和类名。包名规则：如果接口不是public修饰的，接口必须在同一个包下，否则会抛异常。如果接口不是public修饰的，代理类的包名与接口包名相同，否则默认包名为com.sun.proxy。
3. 生成代理类class文件
4. 通过native方法defineClass0方法生成Class对象并加载到JVM。



通过ProxyGenerator.generateProxyClass生成代理类Class写入本地并反编译:

```java
public class Generate {
    public static void main(String[] args) throws IOException {
        byte[] proxyClass = ProxyGenerator.generateProxyClass("$Proxy0", new Class[]{UserService.class});
        FileOutputStream outputStream = new FileOutputStream(new File("d:\\$Proxy0.class"));
        outputStream.write(proxyClass);
        outputStream.flush();
        outputStream.close();
    }
}
```

$Proxy0.java

```java
import com.axuan.aop.proxy.UserService;
import java.lang.reflect.*;

public final class $Proxy0 extends Proxy
    implements UserService
{

    public $Proxy0(InvocationHandler invocationhandler)
    {
        super(invocationhandler);
    }

    public final boolean equals(Object obj)
    {
        try
        {
            return ((Boolean)super.h.invoke(this, m1, new Object[] {
                obj
            })).booleanValue();
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final String toString()
    {
        try
        {
            return (String)super.h.invoke(this, m2, null);
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final void display(String s)
    {
        try
        {
            super.h.invoke(this, m3, new Object[] {
                s
            });
            return;
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final int hashCode()
    {
        try
        {
            return ((Integer)super.h.invoke(this, m0, null)).intValue();
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    static 
    {
        try
        {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] {
                Class.forName("java.lang.Object")
            });
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("com.axuan.aop.proxy.UserService").getMethod("display", new Class[] {
                Class.forName("java.lang.String")
            });
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        }
        catch(NoSuchMethodException nosuchmethodexception)
        {
            throw new NoSuchMethodError(nosuchmethodexception.getMessage());
        }
        catch(ClassNotFoundException classnotfoundexception)
        {
            throw new NoClassDefFoundError(classnotfoundexception.getMessage());
        }
    }
}

```

我们可以看到代理类$Proxy0继承了Proxy类(Java只能单继承)，所以JDK的动态代理只能基于接口。

首先，代理类会通过静态代码块初始化hashCode()、equals()、toString()这三个继承Object的方法，以及实现接口的方法。

```java
  static 
    {
        try
        {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] {
                Class.forName("java.lang.Object")
            });
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("com.axuan.aop.proxy.UserService").getMethod("display", new Class[] {
                Class.forName("java.lang.String")
            });
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        }
        catch(NoSuchMethodException nosuchmethodexception)
        {
            throw new NoSuchMethodError(nosuchmethodexception.getMessage());
        }
        catch(ClassNotFoundException classnotfoundexception)
        {
            throw new NoClassDefFoundError(classnotfoundexception.getMessage());
        }
    }
```

然后，构造方法实例化代理类，入参为InvocationHandler这是父类Proxy的属性`InvocationHandler h`

```java
 public $Proxy0(InvocationHandler invocationhandler)
    {
        super(invocationhandler);
    }

```

最后，覆盖接口方法，方法中调用InvocationHandler的invoke方法，从而实现了动态代理。

```java
  public final void display(String s)
    {
        try
        {
            super.h.invoke(this, m3, new Object[] {
                s
            });
            return;
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }
```



转自:https://www.cnblogs.com/monkey0307/p/8276810.html