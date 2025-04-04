---
description: 设计模式
---

# 代理模式

### 1. **什么是代理模式**

&#x20;       **代理模式**（Proxy Pattern）也称为**委托模式，**&#x662F;结构型设计模式的一种。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。通俗的来讲代理模式就是我们生活中常见的**中介**。

![](<../../../.gitbook/assets/image (252).png>)

&#x20; 代理模式中主要有三个角色，分别是**Subject（目标）**，**RealSubject（真正实现）**，**Proxy（代理）。**

* **Subject：**&#x62BD;象主题类。声明真实主题与代理间的共同的接口方法，可以是抽象类也可以是接口。
* **RealSubject：**&#x771F;实主题类。即被委托类或被代理类，Proxy表示的真实对象，由其具体执行业务逻辑，Client则通过Proxy间接使用该类中定义的方法。
* **Proxy：**&#x4EE3;理类。持有对RealSubject的引用，在其所实现的接口方法中调用RealSubject对应的方法，以此起到代理的作用。

**优点：**

* 职责清晰
* 高扩展性，面向接口编程，业务变化时，场景类无需改变。

**缺点：**

* 由于在客户端和真实主题之间增加了代理对象，因此有些类型的代理模式可能会造成请求的处理速度变慢。&#x20;
* 实现代理模式需要额外的工作，有些代理模式的实现非常复杂。
* 静态代理下，如果业务逻辑变了（Subject 内的方法变了），虽说场景类不需要改变，但是对应的RealSubject，Proxy需要改动，因为这两个是实现了Subject的。

### 2. 静态代理

静态代理示例，律师代理小明劳动仲裁。

Subject类：

```
/**
 * Subject：诉讼接口类
 */
public interface ILawsuit {
    // 提交申请
    void submit();

    // 举证
    void burden();

    // 辩护
    void defend();

    // 诉讼完成
    void finish();
}
```

 RealSubject类：

```
/**
 * 具体诉讼人，小明
 */
public class XiaoMing implements ILawsuit {

    @Override
    public void submit() {
        System.out.println("老板拖欠工资，申请仲裁");
    }

    @Override
    public void burden() {
        System.out.println("这是劳动合同与银行流水");
    }

    @Override
    public void defend() {
        System.out.println("证据确凿");
    }

    @Override
    public void finish() {
        System.out.println("仲裁成功");
    }
}
```

 Proxy类：

```
/**
 * 代理律师
 */
public class Lawyer implements ILawsuit{
    private ILawsuit delegate;

    public Lawyer(ILawsuit delegate) {
        this.delegate = delegate;
    }

    @Override
    public void submit() {
        delegate.submit();
    }

    @Override
    public void burden() {
        delegate.burden();
    }

    @Override
    public void defend() {
        delegate.defend();
    }

    @Override
    public void finish() {
        delegate.finish();
    }
}
```

Client类：

```
/**
 * 客户类，小明找代理律师打官司
 */
public class Client {
    public static void main(String[] args) {
        XiaoMing xiaoMing = new XiaoMing();
        Lawyer lawyer = new Lawyer(xiaoMing);
        lawyer.submit();
        lawyer.burden();
        lawyer.defend();
        lawyer.finish();
    }
}
```

### 3. 动态代理

&#x20;       代理模式根据实现方式的不同可以分成二类，**静态代理与动态代理**。类似上面👆的例子，**代理类的class文件在编译时已经存在的代理称为静态代理。反之，通过反射机制在运行过程中动态的生成代理者对象（也就是说在code阶段压根就不知道代理谁）则称之为动态代理**。

&#x20;       在JDK中给我们提供了一个便捷动态代理接口`InvocationHandler`，通过实现它的invoke方法来调用具体的被代理对象的方法即可实现动态代理。

```
/**
 * 动态代理类
 */
public class DynamicProxy implements InvocationHandler {
    // 被代理对象的引用
    private Object object;

    public DynamicProxy(Object obj) {
        this.object = obj;
    }


    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        method.invoke(object, args);
        return proxy;
    }
}
```

&#x20;       然后通过`Proxy.newProxyInstance`创建动态代理实例，实现代理。

```
/**
 * 客户类，打官司
 */
public class Client {
    public static void main(String[] args) {
        XiaoMing xiaoMing = new XiaoMing();
        DynamicProxy dynamicProxy = new DynamicProxy(xiaoMing);
        ILawsuit lawyer = (ILawsuit) Proxy.newProxyInstance(XiaoMing.class.getClassLoader(),
                new Class[]{ILawsuit.class}, dynamicProxy);
        lawyer.submit();
        lawyer.burden();
        lawyer.defend();
        lawyer.finish();
    }
}
```

**动态代理的实现：**

&#x20;       从`Proxy.newProxyInstance()`的实现跟踪，可发现动态代理的实现是先生成代理类，然后创建其实例，再通过`InvocationHandler` 的回掉方法调用被代理对象的方法。

```
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    /*
     * Look up or generate the designated proxy class.
     */
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
     * Invoke its constructor with the designated invocation handler.
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
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

&#x20;       `newProxyInstance`中`getProxyClass0()`查找/生成目标对象的代理类。`getProxyClass0()`的实现如下。

```
/**
 * Generate a proxy class.  Must call the checkProxyAccess method
 * to perform permission checks before calling this.
 */
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}
```

         代理proxy class最终通过`ProxyClassFactory` 生成。

```
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // 省略不重要的代码。。。

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        // 省略代码。。。

        /*
         * Choose a name for the proxy class to generate.
         */
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        /*
         * Generate the specified proxy class.
         */
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
             * A ClassFormatError here means that (barring bugs in the
             * proxy class generation code) there was some other
             * invalid aspect of the arguments supplied to the proxy
             * class creation (such as virtual machine limitations
             * exceeded).
             */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

         可以看到最终是通过`ProxyGenerator.generateProxyClass` 生成代理的Proxy Class文件。我们以上述demo为例，查看以下`ILawsuit` 的动态代理类生成。

```
private static void generateProxyClass() throws Exception {
    //生成代理指定接口的Class数据
    byte[] bytes = ProxyGenerator.generateProxyClass("ILawsuit$Proxy",
            new Class[]{ILawsuit.class});
    FileOutputStream fos = new FileOutputStream("ILawsuit$Proxy.class");
    fos.write(bytes);
    fos.close();
}
```

         生成的Proxy Class文件如下所示：

```
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

import com.he.ILawsuit;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class ILawsuit$Proxy extends Proxy implements ILawsuit {
    private static Method m1;
    private static Method m6;
    private static Method m2;
    private static Method m5;
    private static Method m3;
    private static Method m4;
    private static Method m0;

    public ILawsuit$Proxy(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void submit() throws  {
        try {
            super.h.invoke(this, m6, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void burden() throws  {
        try {
            super.h.invoke(this, m5, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void finish() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void defend() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m6 = Class.forName("com.he.ILawsuit").getMethod("submit");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m5 = Class.forName("com.he.ILawsuit").getMethod("burden");
            m3 = Class.forName("com.he.ILawsuit").getMethod("finish");
            m4 = Class.forName("com.he.ILawsuit").getMethod("defend");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

         Proxy实现如下：

```
public class Proxy implements java.io.Serializable {

    private static final long serialVersionUID = -2222568056686623797L;

    /** parameter types of a proxy class constructor */
    private static final Class<?>[] constructorParams =
        { InvocationHandler.class };

    /**
     * a cache of proxy classes
     */
    private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());

    /**
     * the invocation handler for this proxy instance.
     * @serial
     */
    protected InvocationHandler h;

    /**
     * Prohibits instantiation.
     */
    private Proxy() {
    
    .....
   
}   
```
