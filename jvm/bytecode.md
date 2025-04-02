# JVM-字节码

## 1. 什么是字节码？

Java之所以可&#x4EE5;**“一次编写，到处运行”**，一是因为JVM针对各种操作系统、平台都进行了定制，二是因为无论在什么平台，都可以编译生成固定格式的字节码（.class文件）供JVM使用。因此，可以看出字节码对于Java生态的重要性。

![、](<../.gitbook/assets/image (130).png>)

Java代码被间接翻译成字节码文件，然后再交由不同平台上的JVM虚拟机去读取执行，从而达到一次编写，到处运行的目的。同时也由此衍生出了许多基于JVM的编程语言，如Groovy，Scala，Koltin等等。

之所以称为字节码，是因为**class文件本质上是一个以8位字节为基础单位的二进制流**，各个部分数据严格按照指定顺序排列在class文件中，相邻的项之间没有间隙（使得class文件非常紧凑，体积轻巧，可以被JVM快速的加载至内存，并且只占据较少的内存空间。字节码文件很灵活，它甚至比Java源文件有着更强的描述能力）。jvm根据其特定的字节码规则解析该二进制数据，从而得到相关信息。Class文件采用一种伪结构来存储数据，它有两种类型：无符号数和表。

## 2. Class文件结构

.java文件通过javac编译后得到一个.class文件，下面是定义的一个空类hello.java生成的class文件。

```
public class Hello { 
    
}
```

![](<../.gitbook/assets/image (5).png>)

根据[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)&#x4E2D;**`The class File Format`** 章节的描述，每一个字节码文件都由指定的部分数据及顺序组成，整体结构如下图所示：

![](<../.gitbook/assets/image (306).png>)

![](<../.gitbook/assets/image (172).png>)

### 2.1 魔数（Magic Number）

所有的.class文件的前四个字节都是魔数，魔数的固定值为：**0xCAFEBABE**。魔数放在文件开头，JVM可以根据文件的开头来判断这个文件是否可能是一个.class文件，如果是，才会继续进行之后的操作。

> 魔数的固定值是Java之父James Gosling制定的，为CafeBabe（咖啡宝贝），而Java的图标为一杯咖啡。

### 2.2 版本号

版本号为魔数之后的4个字节，前两个字节表示次版本号（Minor Version），后两个字节表示主版本号（Major Version）。上图中版本号为“00 00 00 34”，次版本号转化为十进制为0，主版本号转化为十进制为52，在Oracle官网中查询序号52对应的主版本号为1.8，所以编译该文件的Java版本号为1.8.0。

### 2.3 常量池（Constant Pool）

紧接着主版本号之后的字节为常量池入口。常量池可以理解成Class文件中的资源仓库。主要存放的是两大类常量：**字面量(Literal)和符号引用(Symbolic References)**。字面量类似于java中的常量概念，如文本字符串，final常量等，而符号引用则属于编译原理方面的概念，包括以下三种:

* 类和接口的全限定名(Fully Qualified Name)
* 字段的名称和描述符号(Descriptor)
* 方法的名称和描述符

不同于C/C++, JVM是在加载Class文件的时候才进行的动态链接，也就是说这些字段和方法符号引用只有在运行期转换后才能获得真正的内存入口地址。**当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建或运行时解析并翻译到具体的内存地址中**。

常量池整体上分为两部分：**常量池计数器以及常量池数据区**，如下图示：

![](<../.gitbook/assets/image (111).png>)

常量池计数器（constant\_pool\_count）：由于常量的数量不固定，所以需要先放置两个字节来表示常量池容量计数值。

常量池数据区：数据区是由（constant\_pool\_count-1）个cp\_info结构组成，一个cp\_info结构对应一个常量。在字节码中共有14种类型的cp\_info（如下图所示），每种类型的结构都是固定的。

![](<../.gitbook/assets/image (161).png>)

cp\_info整体结构大同小异，都是先通过Tag来标识类型，然后后续n个字节来描述长度和（或）数据。

### 2.4 访问标志

常量池结束之后的两个字节，描述该Class是类还是接口，以及是否被Public、Abstract、Final等修饰符修饰。JVM规范规定了如下图的访问标志（Access\_Flag）。需要注意的是，JVM并没有穷举所有的访问标志，而是使用按位或操作来进行描述的，比如某个类的修饰符为Public Final，则对应的访问修饰符的值为ACC\_PUBLIC | ACC\_FINAL，即0x0001 | 0x0010=0x0011。

| 标志名称            | 标志值    | 含义                                      |
| --------------- | ------ | --------------------------------------- |
| ACC\_PUBLIC     | 0x0001 | 是否为Public类型                             |
| ACC\_FINAL      | 0x0010 | 是否被声明为final，只有类可以设置                     |
| ACC\_SUPER      | 0x0020 | 是否允许使用invokespecial字节码指令的新语义．           |
| ACC\_INTERFACE  | 0x0200 | 标志这是一个接口                                |
| ACC\_ABSTRACT   | 0x0400 | 是否为abstract类型，对于接口或者抽象类来说，次标志值为真，其他类型为假 |
| ACC\_SYNTHETIC  | 0x1000 | 标志这个类并非由用户代码产生                          |
| ACC\_ANNOTATION | 0x2000 | 标志这是一个注解                                |
| ACC\_ENUM       | 0x4000 | 标志这是一个枚举                                |

### 2.5 当前类名

访问标志后的两个字节，描述的是当前类的全限定名。这两个字节保存的值为常量池中的索引值，根据索引值就能在常量池中找到这个类的全限定名。

### 2.6 父类名称

当前类名后的两个字节，描述父类的全限定名，同上，保存的也是常量池中的索引值。

### 2.7 接口信息

父类名称后为两字节的接口计数器，描述了该类或父类实现的接口数量。紧接着的n个字节是所有接口名称的字符串常量的索引值。

### 2.8 字段表

字段表用于描述类和接口中声明的变量，包含**类级别的变量以及实例变量**，但是不包含方法内部声明的局部变量。字段表也分为两部分，第一部分为两个字节，描述字段个数；第二部分是每个字段的详细信息fields\_info。字段表结构如下图所示:

![](<../.gitbook/assets/image (106).png>)

### 2.9 方法表

字段表结束后为方法表，方法表也是由两部分组成，第一部分为两个字节描述方法的个数；第二部分为每个方法的详细信息。方法的详细信息较为复杂，包括方法的访问标志、方法名、方法的描述符以及方法的属性，如下图所示：

![](<../.gitbook/assets/image (185).png>)

方法的权限修饰符依然可以通过上图的值查询得到，方法名和方法的描述符都是常量池中的索引值，可以通过索引值在常量池中找到。而“方法的属性”这一部分较为复杂，直接借助javap -verbose将其反编译为人可以读懂的信息进行解读，如下图所示。可以看到属性中包括以下三个部分：

![](<../.gitbook/assets/image (414).png>)

* **“Code区”：**&#x6E90;代码对应的JVM指令操作码，在**进行字节码增强时重点操作的就是“Code区”**&#x8FD9;一部分。
* **“LineNumberTable”：**&#x884C;号表，将Code区的操作码和源代码中的行号对应，Debug时会起到作用（源代码走一行，需要走多少个JVM指令操作码）。
* **“LocalVariableTable”：**&#x672C;地变量表，包含This和局部变量，之所以可以在每一个方法内部都可以调用This，是因为JVM将This作为每一个方法的第一个参数隐式进行传入。当然，这是针对非Static方法而言。

### 2.10 附加属性表

字节码的最后一部分，该项存放了在该文件中类或接口所定义属性的基本信息。

## 3. 示例

### 3.1 javap的使用

对于字节码的反编译可使用java内置的一个反编译工具javap，其用法: `javap <options> <classes>`其中`<options>`选项包括:

```
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```

### 3.2 示例

代码：

```
public class main {
    private int m;

    public int inc() {
        return m + 1;
    }
}
```

反编译：javap -v -p main.class

```
Classfile /Users/robin/work/kotlin/out/production/kotlin/com/robin/main.class
  Last modified 2021-2-19; size 358 bytes
  MD5 checksum 6d59f17692c31d8e650d74077208eecf
  Compiled from "main.java"
public class com.robin.main
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#18         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#19         // com/robin/main.m:I
   #3 = Class              #20            // com/robin/main
   #4 = Class              #21            // java/lang/Object
   #5 = Utf8               m
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/robin/main;
  #14 = Utf8               inc
  #15 = Utf8               ()I
  #16 = Utf8               SourceFile
  #17 = Utf8               main.java
  #18 = NameAndType        #7:#8          // "<init>":()V
  #19 = NameAndType        #5:#6          // m:I
  #20 = Utf8               com/robin/main
  #21 = Utf8               java/lang/Object
{
  private int m;
    descriptor: I
    flags: ACC_PRIVATE

  public com.robin.main();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/robin/main;

  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/robin/main;
}
SourceFile: "main.java"
```

开头的7行信息包括：Class文件当前所在位置，最后修改时间，文件大小，MD5值，编译自哪个文件，类的全限定名，jdk次版本号，主版本号。

第一个常量是一个方法定义，指向了第4和第18个常量。以此类推查看第4和第18个常量。最后可以拼接成第一个常量右侧的注释内容:

```
java/lang/Object."<init>":()V
```

这段可以理解为该类的实例构造器的声明，由于Main类没有重写构造方法，所以调用的是父类的构造方法。此处也说明了Main类的直接父类是Object。 该方法默认返回值是V, 也就是void，无返回值。

第二个常量同理可得:

```
#2 = Fieldref           #3.#19
#3 = Class              #20  
#5 = Utf8               m
#6 = Utf8               I
#19 = NameAndType        #5:#6          // m:I
#20 = Utf8               com/robin/main
```

此处声明了一个字段m，类型为I，I即是int类型。关于字节码的类型对应如下：

| 标识字符 | 含义                             |
| ---- | ------------------------------ |
| B    | 基本类型byte                       |
| C    | 基本类型char                       |
| D    | 基本类型double                     |
| F    | 基本类型float                      |
| I    | 基本类型int                        |
| J    | 基本类型long                       |
| S    | 基本类型short                      |
| Z    | 基本类型boolean                    |
| V    | 特殊类型void                       |
| L    | 对象类型，以分号结尾，如Ljava/lang/Object; |

对于数组类型，每一位使用一个前置的`[`字符来描述，如定义一个`java.lang.String[][]`类型的维数组，将被记录为`[[Ljava/lang/String;`

```
  private int m;
    descriptor: I
    flags: ACC_PRIVATE
```

此处声明了一个私有变量m，类型为int

```
  public com.robin.main();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/robin/main;
```

这里是构造方法：Main()，返回值为void，公开方法。code内的主要属性为:

* **stack**: 最大操作数栈，JVM运行时会根据这个值来分配栈帧(Frame)中的操作栈深度,此处为1
* **locals**: 局部变量所需的存储空间，单位为Slot, Slot是虚拟机为局部变量分配内存时所使用的最小单位，为4个字节大小。方法参数(包括实例方法中的隐藏参数this)，显示异常处理器的参数(try catch中的catch块所定义的异常)，方法体中定义的局部变量都需要使用局部变量表来存放。值得一提的是，locals的大小并不一定等于所有局部变量所占的Slot之和，因为局部变量中的Slot是可以重用的。
* **args\_size**: 方法参数的个数，这里是1，因为每个实例方法都会有一个隐藏参数this
* **attribute\_info**: 方法体内容，0,1,4为字节码"行号"，该段代码的意思是将第一个引用类型本地变量推送至栈顶，然后执行该类型的实例方法，也就是常量池存放的第一个变量，也就是注释里的"java/lang/Object."")V", 然后执行返回语句，结束方法。
* **LineNumberTable**: 该属性的作用是描述源码行号与字节码行号(字节码偏移量)之间的对应关系。可以使用 -g:none 或-g:lines选项来取消或要求生成这项信息，如果选择不生成LineNumberTable，当程序运行异常时将无法获取到发生异常的源码行号，也无法按照源码的行数来调试程序。
* **LocalVariableTable**: 该属性的作用是描述帧栈中局部变量与源码中定义的变量之间的关系。可以使用 -g:none 或 -g:vars来取消或生成这项信息，如果没有生成这项信息，那么当别人引用这个方法时，将无法获取到参数名称，取而代之的是arg0, arg1这样的占位符。 start 表示该局部变量在哪一行开始可见，length表示可见行数，Slot代表所在帧栈位置，Name是变量名称，然后是类型签名。

同理可以分析Main类中的另一个方法"inc()":

```
  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/robin/main;

```

方法体内的内容是：将this入栈，获取字段#2并置于栈顶, 将int类型的1入栈，将栈内顶部的两个数值相加，返回一个int类型的值。

## 4. 字节码增强



## 5. 参考

* [**The Class File Format**](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)
* [**字节码增强技术探索**](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)



