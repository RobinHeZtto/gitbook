# Java反射

### &#x20;1. 什么是反射

&#x20;       反射机制指的是程序在**运行时能够获取自身的信息**。在java中，只要给定类的名字，那么就可以通过反射机制来获得类的所有属性和方法。

&#x20;       反射的作用：

* 在运行时判断任意一个对象所属的类。
* 在运行时判断任意一个类所具有的成员变量和方法。
* 在运行时任意调用一个对象的方法。
* 在运行时构造任意一个类的对象。

&#x20;       反射是Java中一种强大的工具，能够使我们很方便的创建灵活的代码，这些代码可以在运行时装配，无需在组件之间进行源代码链接。但是反射使用不当会成本很高！类中有什么信息，利用反射机制就能可以获得什么信息，不过前提是得知道类的名字。

### 2. 反射的优缺点

&#x20;       **反射机制的优点**：可以实现动态创建对象和编译，体现出很大的灵活性（特别是在J2EE的开发中它的灵活性就表现的十分明显）。通过反射机制我们可以获得类的各种内容，进行反编译。对于JAVA这种先编译再运行的语言来说，反射机制可以使代码更加灵活，更加容易实现面向对象。

&#x20;       比如，一个大型的软件，不可能一次就把把它设计得很完美，把这个程序编译后，发布了，当发现需要更新某些功能时，我们不可能要用户把以前的卸载，再重新安装新的版本，假如这样的话，这个软件肯定是没有多少人用的。采用静态的话，需要把整个程序重新编译一次才可以实现功能的更新，而采用反射机制的话，它就可以不用卸载，只需要在运行时动态地创建和编译，就可以实现该功能。

&#x20;       **反射机制的缺点**：

* **性能第一** \
  反射包括了一些动态类型，所以JVM无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被 执行的代码或对性能要求很高的程序中使用反射。
* **安全限制** \
  使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如Applet，那么这就是个问题了。
* **内部暴露**  \
  由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用－－代码有功能上的错误，降低可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

### 3. Class类

#### &#x20; Class对象是什么？     &#x20;

&#x20;       **`Java.lang.Class`类是java反射机制的基础**，通过`Class`类我们可以获得关于一个类的相关信息。它是一个比较特殊的类，用于封装被装入到JVM中的类（包括类和接口）的信息，通过它可以对被装入类的详细信息进行访问。每个类（型）都有自己的Class对象，如果说类是对象抽象和集合的话，那么Class类就是对类的抽象和集合。

&#x20;        Class 类没有公共的构造方法，**Class对象是在类加载的时候由Java虚拟机以及通过调用类加载器中的 defineClass 方法自动构造的**，因此不能显式地声明一个Class对象。一个类被加载到内存并供我们使用需要经历如下三个阶段：

1. **加载。**&#x8FD9;是由类加载器（ClassLoader）执行的。通过一个类的全限定名来获取其定义的二进制字节流（Class字节码），将这个字节流所代表的静态存储结构转化为方法区的运行时数据接口，根据字节码在java堆中生成一个代表这个类的java.lang.Class对象。
2. **链接**。在链接阶段将验证Class文件中的字节流包含的信息是否符合当前虚拟机的要求，为静态域分配存储空间并设置类变量的初始值（默认的零值），并且如果必需的话，将常量池中的符号引用转化为直接引用。
3. **初始化**。到了此阶段，才真正开始执行类中定义的java程序代码。用于执行该类的静态初始器和静态初始块，如果该类有父类的话，则优先对其父类进行初始化。

#### **获取Class对象的方式**

* Class.forName(“类的全限定名”)
* 实例对象.getClass()
* 类名.class （类字面常量）

&#x20;        Class.forName()除了将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块。注意这里的静态块指的是在类初始化时的一些数据，而类名.class不会初始化。

**注意⚠️：**

&#x20;       `int.class`与`Integer.class`并不是相同的对象，`int.class`等价于`Integer.Type`，`Integer.Type`定义如下：

```
public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
```

&#x20;       基本类型与包装类型Class的对应关系如下：

![](<../../.gitbook/assets/image (153).png>)

#### Class类的主要方法

&#x20;       先说明一下，**构造方法都是获取本类的(一定不包括父类)，只是公共和其他修饰符的区别**。以下其他带有 `Declared` 的方法，都表示**只能获取本类的属性，**&#x4E0D;包括父类的，不过可以获取所有修饰符修饰的属性(**公有，保护，默认，私有**)。不带有 `Declared` 的方法，可以**获取本类及父类**的所有的属性，只能获取 **公有** 修饰符的 。

&#x20; 1\. 获取构造函数：

```
// 返回类中所有的public构造器集合，默认构造器的下标为0
public Constructor<?>[] getConstructors()
// 返回指定public构造器，参数为构造器参数类型集合
public Constructor<T> getConstructor(Class<?>... parameterTypes)

// 返回类中所有的构造器，包括私有
public Constructor<?>[] getDeclaredConstructors()
// 返回类中指定的构造器，包括私有，参数为构造器参数类型集合
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)

Constructor中的重要方法：
// 实例化类
public T newInstance(Object ... initargs)
```

&#x20;  2\. 获取成员变量：

```
public Field getField(String name)
public Field[] getFields() 

public Field getDeclaredField(String name)
public Field[] getDeclaredFields()

Field中的重要方法：
// 设置私有属性是否可访问；
public void setAccessible(boolean flag)
// 设置字段值
public void set(Object obj, Object value)
```

&#x20;       获取成员方法 ：

```
public Method getMethod(String name, Class<?>... parameterTypes)
public Method getDeclaredMethod(String name, Class<?>... parameterTypes)

public Method[] getMethods()
public Method[] getDeclaredMethods()

Method中重要的方法：
// 执行对象的方法，第一个参数为类实例对象，第二个参数：对象方法的参数。
public Object invoke(Object obj, Object... args)  
```

         获取注解方法：

```
public Annotation[] getAnnotations()
public <A extends Annotation> A getAnnotation(Class<A> annotationClass)
public AnnotatedType[] getAnnotatedInterfaces() 
```

&#x20;     实例化对象：

```
public T newInstance()
```

&#x20;       获取类名/包名信息：

```
public String getName()
public String getCanonicalName()
public String getSimpleName()
public Package getPackage()
```



[\
](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/usage-of-reflection)
