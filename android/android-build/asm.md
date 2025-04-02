# ASM详解

## ASM介绍

什么是ASM？ASM是一个java字节码操纵框架，它能被用来动态生成类或者增强既有类的功能。ASM 可以直接产生二进制 class 文件，也可以在类被加载入 Java 虚拟机之前动态改变类行为。

在Android中，ASM一般搭配Gradle Plugin在TransfromApi中使用。\


## ASM使用

ASM Core API提供了3个类来操作字节码，分别是：

* `ClassReader`:对具体的`class`文件进行读取与解析；
* `ClassWriter`:将修改后的`class`文件通过文件流的方式覆盖掉原来的`class`文件，从而实现`class`修改；
* `ClassVisitor`:可以访问`class`文件的各个部分，比如`方法`、`变量`、`注解`等，这也是修改原代码的地方。

注意 ⚠️：\
&#x20;`ClassReader`解析`class`文件过程中，解析到某个结构就会通知到`ClassVisitor`内部的相应方法（比如：解析到方法时，就会回调`ClassVisitor.visitMethod`方法）；



invokestatic：调用静态方法；

invokespecial：调用实例构造方法\<init>，私有方法和父类方法；

invokevirtual：调用虚方法；

invokeinterface：调用接口方法，在运行时再确定一个实现此接口的对象；

invokedynamic：在运行时动态解析出调用点限定符所引用的方法之后，调用该方法；\


```
ClassReader classReader = new ClassReader(is);
//COMPUTE_MAXS 说明使用ASM自动计算本地变量表最大值和操作数栈的最大值
ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
//EXPAND_FRAMES 说明在读取 class 的时候同时展开栈映射帧(StackMap Frame)，
//在使用 AdviceAdapter里这项是必须打开的
classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
```

## 参考

* [字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)
