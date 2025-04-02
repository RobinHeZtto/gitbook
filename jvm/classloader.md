# JVM-类加载器

## 1. 什么是类加载器？

顾名思义，**类加载器（classloader）**&#x662F;Java运行时环境（Java Runtime Environment）的一个部件，负责动态加载Java类到Java虚拟机的内存空间中，并且在加载过程中对数据进行**校验、转换解析和初始化**，最终根据Class文件信息生成与之对应的class对象。

Class文件由类装载器装载后，在JVM中将形成一份**描述Class结构的元信息对象**，通过该元信息对象可以获取Class的结构信息：如构造函数，属性和方法等，即我们经常使用的Class类。

## 2. 类加载过程

类加载器把一个类加载到JVM内存空间，要经过以下步骤：

![](<../.gitbook/assets/image (422).png>)

![](<../.gitbook/assets/image (216).png>)

**加载：**&#x67E5;找和导入Class文件；

**链接：**&#x628A;类的二进制数据合并到JRE中；

* **校验：**&#x68C0;查载入Class文件数据的正确性；
* **准备：**&#x7ED9;类的静态变量分配存储空间；
* **解析：**&#x5C06;符号引用转成直接引用；

**初始化：**&#x5BF9;类的静态变量，静态代码块执行初始化操作

### 2.1 类加载条件

加载是类加载过程中的第一个阶段。有两种时机会触发类加载：

* **预加载。**&#x865A;拟机启动时加载，比如：Android系统中preloaded-classes的文件路径：frameworks/base/preloaded-classes；一般JVM虚拟机启动时加载是JAVA\_HOME/lib/下的rt.jar下的.class文件，这个jar包里面的内容是程序运行时非常常常用到的，像**java.lang.\*、java.util.\*、java.io.\***&#x7B49;。
* **运行时加载。**&#x865A;拟机在用到一个.class文件的时候，会先去内存中查看一下这个.class文件有没有被加载，如果没有就会按照类的全限定名来加载这个类 \
  \- 当创建一个类的实例时，比如使用 new 关键字或者反射、克隆、反序列化\
  \- 当调用类的静态方法时，即使用字节码 invodestatic 指令\
  \- 当使用类或接口的静态字段时（final 常量除外），比如使用 getstatic 或者 putstatic 指令\
  \- 当使用 java.lang.reflect 包中的方法反射类的方法时\
  \- 当初始化子类时，要求先初始化父类\
  \- 作为启动虚拟机，含有 main() 方法的那个类

### **2.2 加载**

类的加载装载指的是将类的class文件中的二进制数据读入到内存中，**将其放在运行时数据区的方法区(metadata)内，然后在堆区创建一个java.lang.Class对象**，用来封装类在方法区内的数据结构。类的加载的最终结果是位于堆区中的Class对象，**Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口**。

![](<../.gitbook/assets/image (120).png>)

类加载器并不需要等到某个类被“首次主动使用”时再加载它，JVM规范允许类加载器在预料某个类将要被使用时就预先加载它，如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误（LinkageError错误）如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误。

加载class文件的方式有:

* 从本地系统中直接加载&#x20;
* 通过网络下载.class文件&#x20;
* 从zip，jar等归档文件中加载.class文件&#x20;
* 从专有数据库中提取.class文件&#x20;
* 将Java源文件动态编译为.class文件

### **2.3 验证**

验证的目的是为了**确保Class文件中的字节流包含的信息符合当前虚拟机的要求，而且不会危害虚拟机自身的安全**。不同的虚拟机对类验证的实现可能会有所不同，但大致都会完成以下四个阶段的验证：**文件格式的验证、元数据的验证、字节码验证和符号引用验证**。

1）文件格式的验证：验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理，该验证的主要目的是保证输入的字节流能正确地解析并存储于方法区之内。经过该阶段的验证后，字节流才会进入内存的方法区中进行存储，后面的三个验证都是基于方法区的存储结构进行的。

2）元数据验证：对类的元数据信息进行语义校验（其实就是对类中的各数据类型进行语法校验），保证不存在不符合Java语法规范的元数据信息。

3）字节码验证：该阶段验证的主要工作是进行数据流和控制流分析，对类的方法体进行校验分析，以保证被校验的类的方法在运行时不会做出危害虚拟机安全的行为。

4）符号引用验证：这是最后一个阶段的验证，它**发生在虚拟机将符号引用转化为直接引用的时候**（解析阶段中发生该转化，后面会有讲解），主要是对类自身以外的信息（常量池中的各种符号引用）进行匹配性的校验。

### **2.4 准备**

准备阶段是**正式为类变量分配内存并设置类变量初始值**的阶段，**这些内存都将在方法区中进行分配**。 注意⚠️：

* 这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
* 这里所设置的初始值通常情况下是**数据类型默认的零值**（如0、0L、null、false等），而不是被在Java代码中被显式地赋予的值。

### **2.5 解析**

解析阶段是**虚拟机将常量池内的符号引用替换为直接引用**的过程。

**符号引用（Symbolic Reference）**：符号引用以一组符号来描述所引用的目标，符号引用可以是任何形式的字面量，符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经在内存中。

**直接引用（Direct Reference）**：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是与虚拟机实现的内存布局相关的，同一个符号引用在不同的虚拟机实例上翻译出来的直接引用一般都不相同，如果有了直接引用，那引用的目标必定已经在内存中存在。

1\)、类或接口的解析：判断所要转化成的直接引用是对数组类型，还是普通的对象类型的引用，从而进行不同的解析。&#x20;

2\)、字段解析：对字段进行解析时，会先在本类中查找是否包含有简单名称和字段描述符都与目标相匹配的字段，如果有，则查找结束；如果没有，则会按照继承关系从上往下递归搜索该类所实现的各个接口和它们的父接口，还没有，则按照继承关系从上往下递归搜索其父类，直至查找结束。

3\)、类方法解析：对类方法的解析与对字段解析的搜索步骤差不多，只是多了判断该方法所处的是类还是接口的步骤，而且对类方法的匹配搜索，是先搜索父类，再搜索接口。

4\)、接口方法解析：与类方法解析步骤类似，只是接口不会有父类，因此，只递归向上搜索父接口就行了。

### **2.6 初始化**

类初始化阶段是类加载过程的最后一步，前面的类加载过程中，除了加载（Loading）阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码。 **初始化，为类的静态变量赋予正确的初始值，JVM负责对类进行初始化，主要对类变量进行初始化**。在Java中对类变量进行初始值设定有两种方式：

1. 声明类变量时指定初始值
2. 使用静态代码块为类变量指定初始值

JVM初始化步骤:

* 假如这个类还没有被加载和连接，则程序先加载并连接该类
* 假如该类的直接父类还没有被初始化，则先初始化其直接父类
* 假如类中有初始化语句，则系统依次执行这些初始化语句

初始化阶段时执行类构造器方法的过程:

1）类构造器方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序由语句在源文件中出现的顺序所决定。

2）类构造器方法与类的构造函数不同，它不需要显式地调用父类构造器，虚拟机会保证在子类的类构造器方法执行之前，父类的类构造器方法已经执行完毕，因此在虚拟机中第一个执行的类构造器方法的类一定是java.lang.Object。

3）由于父类的类构造器方法方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作。

4）类构造器方法对于类或者接口来说并不是必需的，如果一个类中没有静态语句块也没有对变量的赋值操作，那么编译器可以不为这个类生成类构造器方法。

5）接口中可能会有变量赋值操作，因此接口也会生成类构造器方法。但是接口与类不同，执行接口的类构造器方法不需要先执行父接口的类构造器方法。只有当父接口中定义的变量被使用时，父接口才会被初始化。另外，接口的实现类在初始化时也不会执行接口的类构造器方法。

6）虚拟机会保证一个类的类构造器方法在多线程环境中被正确地加锁和同步。如果有多个线程去同时初始化一个类，那么只会有一个线程去执行这个类的类构造器方法，其它线程都需要阻塞等待，直到活动线程执行类构造器方法完毕。如果在一个类的类构造器方法中有耗时很长的操作，那么就可能造成多个进程阻塞。

## 3. 双亲委派

类加载器的层次结构：

![](<../.gitbook/assets/image (144).png>)

> 注意⚠️：这里的层次结构并非是继承关系！

**从虚拟机的角度来说：**&#x53EA;存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），该类加载器使用C++语言实现，属于虚拟机自身的一部分。另外一种就是所有其它的类加载器，这些类加载器是由Java语言实现，独立于JVM外部，并且全部继承自抽象类java.lang.ClassLoader。

**从Java开发人员的角度来看：**&#x5927;部分Java程序一般会使用到以下三种系统提供的类加载器：

* **启动类加载器（Bootstrap ClassLoader）**：负责加载JAVA\_HOME\lib目录中并且能被虚拟机识别的类库到JVM内存中，如果名称不符合的类库即使放在lib目录中也不会被加载。该类加载器无法被Java程序直接引用。
* **扩展类加载器（Extension ClassLoader）**：该加载器主要是负责加载JAVA\_HOME\lib\，该加载器可以被开发者直接使用。
* **应用程序类加载器（Application ClassLoader）**：该类加载器也称为系统类加载器，它负责加载用户类路径（Classpath）上所指定的类库，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

双亲委派模型的工作过程为：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的加载器都是如此，因此所有的类加载请求都会传给顶层的启动类加载器，只有当父加载器反馈自己无法完成该加载请求（该加载器的搜索范围中没有找到对应的类）时，子加载器才会尝试自己去加载。

![](<../.gitbook/assets/image (166).png>)

使用这种模型来组织类加载器之间的关系的好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如java.lang.Object类，无论哪个类加载器去加载该类，最终都是由启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。否则的话，如果不使用该模型的话，如果用户自定义一个java.lang.Object类且存放在classpath中，那么系统中将会出现多个Object类，应用程序也会变得很混乱。如果我们自定义一个rt.jar中已有类的同名Java类，会发现JVM可以正常编译，但该类永远无法被加载运行。

```
public Class<?> loadClass(String name)throws ClassNotFoundException {
            return loadClass(name, false);
    }
    protected synchronized Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException {
            // 首先判断该类型是否已经被加载
            Class c = findLoadedClass(name);
            if (c == null) {
                //如果没有被加载，就委托给父类加载或者委派给启动类加载器加载
                try {
                    if (parent != null) {
                         //如果存在父类加载器，就委派给父类加载器加载
                        c = parent.loadClass(name, false);
                    } else {
                    //如果不存在父类加载器，就检查是否是由启动类加载器加载的类，通过调用本地方法native Class findBootstrapClass(String name)
                        c = findBootstrapClass0(name);
                    }
                } catch (ClassNotFoundException e) {
                 // 如果父类加载器和启动类加载器都不能完成加载任务，才调用自身的加载功能
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
```

通过上面代码可以看出，双亲委派模型是通过loadClass()方法来实现的，根据代码以及代码中的注释可以很清楚地了解整个过程其实非常简单：先检查是否已经被加载过，如果没有则调用父加载器的loadClass()方法，如果父加载器为空则默认使用启动类加载器作为父加载器。如果父类加载器加载失败，则先抛出ClassNotFoundException，然后再调用自己的findClass()方法进行加载。

![](<../.gitbook/assets/image (250).png>)

**双亲委派优势：**

* 系统类防止内存中出现多份同样的字节码
* 保证Java程序安全稳定运行

## 4. 自定义类加载器

通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。比如应用是通过网络来传输 Java 类的字节码，为保证安全性，这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现。自定义类加载器一般都是继承自 ClassLoader 类，从上面对 loadClass 方法来分析来看，我们只需要重写 findClass 方法即可。下面我们通过一个示例来演示自定义类加载器的流程:

```
package com.pdai.jvm.classloader;
import java.io.*;

public class MyClassLoader extends ClassLoader {

    private String root;

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] loadClassData(String className) {
        String fileName = root + File.separatorChar
                + className.replace('.', File.separatorChar) + ".class";
        try {
            InputStream ins = new FileInputStream(fileName);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0;
            while ((length = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, length);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    public String getRoot() {
        return root;
    }

    public void setRoot(String root) {
        this.root = root;
    }

    public static void main(String[] args)  {

        MyClassLoader classLoader = new MyClassLoader();
        classLoader.setRoot("D:\\temp");

        Class<?> testClass = null;
        try {
            testClass = classLoader.loadClass("com.pdai.jvm.classloader.Test2");
            Object object = testClass.newInstance();
            System.out.println(object.getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}

```

自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在该类中对文件进行解密。由于这里只是演示，我并未对class文件进行加密，因此没有解密的过程。

**这里有几点需要注意** :

1、这里传递的文件名需要是类的全限定性名称，即`com.pdai.jvm.classloader.Test2`格式的，因为 defineClass 方法是按这种格式进行处理的。

2、最好不要重写loadClass方法，因为这样容易破坏双亲委托模式。

3、这类Test 类本身可以被 AppClassLoader 类加载，因此我们不能把com/pdai/jvm/classloader/Test2.class 放在类路径下。否则，由于双亲委托机制的存在，会直接导致该类由 AppClassLoader 加载，而不会通过我们自定义类加载器来加载。
