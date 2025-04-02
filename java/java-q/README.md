# Java基础

### Q0. 面向对象与面向过程的区别？

**面向过程：自顶而下的编程模式。**

&#x20;       **将问题拆解成一个个步骤，每个步骤通过不同的函数实现，然后依次调用。**&#x5C31;是说，在进行面向过程编程的时候，不需要考虑那么多，上来先定义一个函数，然后使用各种诸如if-else、for-each等方式进行代码执行。最典型的用法就是实现一个简单的算法，比如实现冒泡排序。

**面向对象：将事物高度抽象化的编程模式。**

&#x20;       **将问题分解抽象成一个个对象，通过不同对象之间的调用、组合解决问题。**&#x5C31;是说，在进行面向对象进行编程的时候，要把属性、行为等封装成对象，然后基于这些对象及对象的能力进行业务逻辑的实现。比如:想要造一辆车，上来要先把车的各种属性定义出来，然后抽象成一个Car类。

**优劣对比：**

&#x20;       面向对象:占用资源相对高,速度相对慢

&#x20;       面向过程:占用资源相对低,速度相对快

### Q1. 谈谈你对Java平台的理解？

&#x20;       Java是一种面向对象的语言，有二个显著的特性。其一是**一次编写，到处执行（“Write once， Run anyway“）的跨平台特性**。其二，**自动垃圾回收机制**，通过垃圾收集器（Garbage Collector）自动回收分配内存，大部分情况下，程序员不需要自己操心内存的分配和回收。

![](<../../.gitbook/assets/image (275).png>)

&#x20;       Java的平台无关性的支持是分布在整个Java体系结构中的。其中扮演着重要角色的有Java语言规范、Class文件、Java虚拟机等。

* **Java语言规范**
  * 通过规定Java语言中基本数据类型的取值范围和行为
* **Class文件**
  * 所有Java文件要编译成统一的Class文件
* **Java虚拟机**
  * 通过Java虚拟机将Class文件转成对应平台的二进制文件等，Java虚拟机是平台相关的，通过它屏蔽了底层操作系统和硬件的差异。

> JVM并不是和Java文件直接进行交互的，而是和Class文件，也就是说，其实JVM运行的时候，并不依赖于Java语言。除Java语言之外，如Groovy、Scala、Jython，Kotlin也都可以被编译成字节码运行在JVM上。

### Q2. Java是解释执行的吗？

不完全正确，Java是编译与解释混合执行的。

![](<../../.gitbook/assets/image (6).png>)

&#x20;       通常情况下，java代码首先通过javac被编译成class字节码，然后由Java虚拟机内嵌的解释器逐行将字节码转化成机器码执行。但是现在的JVM为了执行效率，通常都提供了**JIT（Just-In-Time）**&#x7F16;译器，JIT编译器能够在运行时将热点代码直接转化成机器码（根据二八定律，消耗大部分系统资源的只是一小部分热点代码），这种情况下，这部分热点代码的执行就是编译执行，而不是解释执行。

&#x20;       Java 9引入了**AOT（Ahead of Time Compilation）**，能够将**class文件直接编译成可执行二进制文件**，这样就避免了JIT运行时预热等各方面的开销。

⚠&#xFE0F;_: JIT为方法级，他会缓存编译过的字节码在CodeCache中，而不需要重复编译。_

**JIT VS AOT：**

JIT的优点：

1. 可以根据当前硬件情况及运行情况实时编译生成最优机器指令（ps. AOT也可以做到，在用户使用是使用字节码根据机器情况在做一次编译）
2. 当程序需要支持动态链接时，只能使用JIT
3. 可以根据进程中内存的实际情况调整代码，使内存能够更充分的利用

JIT的缺点：

1. 编译需要占用运行时资源，可能会导致进程卡顿，对于某些代码的编译优化不能完全支持，需要在程序流畅和编译时间之间做权衡。且在编译准备和识别频繁使用的方法需要占用时间，使得初始编译不能达到最高性能。

AOT的优点：

1. 在程序运行前编译，可以避免在运行时的编译性能消耗和内存消耗，可以在程序运行初期就达到最高性能，显著的加快程序的启动。

AOT的缺点：

1. 在程序运行前编译会使程序安装的时间增加，并且牺牲Java的一致性，提前编译的内容会占用更多的空间。

![](<../../.gitbook/assets/image (478).png>)



### Q3. 面向对象的三大基本特征？

#### 封装(Encapsulation) <a href="#feng-zhuang-encapsulation" id="feng-zhuang-encapsulation"></a>

&#x20;       所谓封装，也就是把客观事物封装成抽象的类，并且类可以把自己的数据和方法只让可信的类或者对象操作，对不可信的进行信息隐藏。封装是面向对象的特征之一，是对象和类概念的主要特性。简单的说，一个类就是一个封装了数据以及操作这些数据的代码的逻辑实体。在一个对象内部，某些代码或某些数据可以是私有的，不能被外界访问。通过这种方式，对象对内部数据提供了不同级别的保护，以防止程序中无关的部分意外的改变或错误的使用了对象的私有部分。

#### 继承(Inheritance) <a href="#ji-cheng-inheritance" id="ji-cheng-inheritance"></a>

&#x20;       继承是指这样一种能力：它可以使用现有类的所有功能，并在无需重新编写原来的类的情况下对这些功能进行扩展。通过继承创建的新类称为“子类”或“派生类”，被继承的类称为“基类”、“父类”或“超类”。继承的过程，就是从一般到特殊的过程。要实现继承，可以通过“继承”（Inheritance）和“组合”（Composition）来实现。继承概念的实现方式有二类：实现继承与接口继承。实现继承是指直接使用基类的属性和方法而无需额外编码的能力；接口继承是指仅使用属性和方法的名称、但是子类必须提供实现的能力；

#### 多态(Polymorphism) <a href="#duo-tai-polymorphism" id="duo-tai-polymorphism"></a>

&#x20;       所谓多态就是指一个类实例的相同方法在不同情形有不同表现形式。多态机制使具有不同内部结构的对象可以共享相同的外部接口。这意味着，虽然针对不同对象的具体操作不同，但通过一个公共的类，它们（那些操作）可以通过相同的方式予以调用。最常见的多态就是将子类传入父类参数中，运行时调用父类方法时通过传入的子类决定具体的内部结构或行为。

### Q4. 多态的意义？

&#x20;       多态是指不同类的对象对同一消息做出响应。即同一消息可以根据发送对象的不同而表现出多种不同的行为方式。多态存在的三个必要条件： **要有继承**、**要有重写**、**父类引用指向子类对象**。多态是通过动态绑定（dynamic binding）的方式实现，在执行期间判断所引用对象的实际类型调用对应的方法。多态的作用是为了消除类型之间的耦合关系。它的好处：

* 1.可替换性（substitutability）。多态对已存在代码具有可替换性。例如，多态对圆Circle类工作，对其他任何圆形几何体，如圆环，也同样工作。&#x20;
* 2.可扩充性（extensibility）。多态对代码具有可扩充性。增加新的子类不影响已存在类的多态性、继承性，以及其他特性的运行和操作。实际上新加子类更容易获得多态功能。例如，在实现了圆锥、半圆锥以及半球体的多态基础上，很容易增添球体类的多态性。&#x20;
* 3.接口性（interface-ability）。多态是超类通过方法签名，向子类提供了一个共同接口，由子类来完善或者覆盖它而实现的。
* 4.灵活性（flexibility）。它在应用中体现了灵活多样的操作，提高了使用效率。&#x20;
* 5.简化性（simplicity）。多态简化对应用软件的代码编写和修改过程，尤其在处理大量对象的运算和操作时，这个特点尤为突出和重要。

> 另外，还有一种说法，包括维基百科也说明，多态还分为动态多态和静态多态。上面提到的那种动态绑定认为是动态多态，因为只有在运行期才能知道真正调用的是哪个类的方法。
>
> 还有一种静态多态，一般认为Java中的函数重载是一种静态多态，因为他需要在编译期决定具体调用哪个方法。关于这个动态静态的说法，我更偏向于重载和多态其实是无关的。但是也要看情况，普通场合，我会认为只有方法的重写算是多态，毕竟这是我的观点。但是如果在面试的时候，我“可能”会认为重载也算是多态，毕竟面试官也有他的观点。我会和面试官说：我认为，多态应该是一种运行期特性，Java中的重写是多态的体现。不过也有人提出重载是一种静态多态的想法，这个问题在StackOverflow等网站上有很多人讨论，但是并没有什么定论。我更加倾向于重载不是多态。

> 在Java中，通过使用一小段特殊的代码来代替绝对地址的调用实现动态绑定，这段代码使用对象中存储的信息来计算方法体的地址。Java的动态绑定是默认的行为，而C++需要使用关键子virtual实现。



### Q5. 重写与重载的区别？

**重载：**&#x51FD;数或者方法有同样的名称，但是参数列表不相同的情形，这样的同名不同参数的函数或者方法之间，互相称之为重载函数或者方法。

**重写：**&#x6307;的是在Java的子类与父类中有两个名称、参数列表都相同的方法的情况。由于他们具有相同的方法签名，所以子类中的新方法将覆盖父类中原有的方法。

**重载 VS 重写：**

关于重载和重写，你应该知道以下几点：

> 1、重载是一个编译期概念、重写是一个运行期间概念。
>
> 2、重载遵循所谓“编译期绑定”，即在编译时根据参数变量的类型判断应该调用哪个方法。
>
> 3、重写遵循所谓“运行期绑定”，即在运行的时候，根据引用变量所指向的实际对象的类型来调用方法
>
> 4、因为在编译期已经确定调用哪个方法，所以重载并不是多态。而重写是多态。重载只是一种语言特性，是一种语法规则，与多态无关，与面向对象也无关。（注：严格来说，重载是编译时多态，即静态多态。但是，Java中提到的多态，在不特别说明的情况下都指动态多态）

**重写的条件：**

* 参数列表必须完全与被重写方法的相同；
* 返回类型必须完全与被重写方法的返回类型相同；
* 访问级别的限制性一定不能比被重写方法的强；
* 访问级别的限制性可以比被重写方法的弱；
* 重写方法一定不能抛出新的检查异常或比被重写的方法声明的检查异常更广泛的检查异常
* 重写的方法能够抛出更少或更有限的异常（也就是说，被重写的方法声明了异常，但重写的方法可以什么也不声明）
* 不能重写被标示为final的方法；
* 如果不能继承一个方法，则不能重写这个方法。

**重载的条件：**

* 被重载的方法必须改变参数列表；
* 被重载的方法可以改变返回类型；
* 被重载的方法可以改变访问修饰符；
* 被重载的方法可以声明新的或更广的检查异常；
* 方法能够在同一个类中或者在一个子类中被重载；

### Q6. 继承 VS 组合

&#x20;       继承（Inheritance）是一种联结类与类的层次模型。指的是一个类（称为子类、子接口）继承另外的一个类（称为父类、父接口）的功能，并可以增加它自己的新功能的能力，继承是类与类或者接口与接口之间最常见的关系；继承是一种[`is-a`](https://zh.wikipedia.org/wiki/Is-a)关系。

![](<../../.gitbook/assets/image (199).png>)

&#x20;       组合(Composition)体现的是整体与部分、拥有的关系，即[`has-a`](https://en.wikipedia.org/wiki/Has-a)的关系。

![](<../../.gitbook/assets/image (258).png>)

继承与组合的优缺点对比：

| 组 合 关 系                          | 继 承 关 系                                |
| -------------------------------- | -------------------------------------- |
| 优点：不破坏封装，整体类与局部类之间松耦合，彼此相对独立     | 缺点：破坏封装，子类与父类之间紧密耦合，子类依赖于父类的实现，子类缺乏独立性 |
| 优点：具有较好的可扩展性                     | 缺点：支持扩展，但是往往以增加系统结构的复杂度为代价             |
| 优点：支持动态组合。在运行时，整体对象可以选择不同类型的局部对象 | 缺点：不支持动态继承。在运行时，子类无法选择不同的父类            |
| 优点：整体类可以对局部类进行包装，封装局部类的接口，提供新的接口 | 缺点：子类不能改变父类的接口                         |
| 缺点：整体类不能自动获得和局部类同样的接口            | 优点：子类能自动继承父类的接口                        |
| 缺点：创建整体类的对象时，需要创建所有局部类的对象        | 优点：创建子类的对象时，无须创建父类的对象                  |

**同样可行的情况下，优先使用组合，组合确实比继承更加灵活，也更有助于代码维护。**

> 最清晰的判断方法是是否需要从新类向基类向上转型，如果需要，继承是必要的。如果不需要，则需要好好考虑下使用继承的理由。

### **Q7. 值传递 VS 引用传递**

&#x20;       **值传递**（pass by value）是指在调用函数时将实际参数`复制`一份传递到函数中，这样在函数中如果对`参数`进行修改，将不会影响到实际参数。

&#x20;       **引用传递**（pass by reference）是指在调用函数时将实际参数的地址`直接`传递到函数中，那么在函数中对`参数`所进行的修改，将影响到实际参数。

![](<../../.gitbook/assets/image (455).png>)

**为什么说java中只有值传递？**

&#x20;       _错误理解一：值传递和引用传递，区分的条件是传递的内容，如果是个值，就是值传递。如果是个引用，就是引用传递。_

&#x20;       _错误理解二：Java是引用传递。_

&#x20;       _错误理解三：传递的参数如果是普通类型，那就是值传递，如果是对象，那就是引用传递。_

&#x20;       我们说当进行方法调用的时候，需要把实际参数传递给形式参数，那么传递的过程中到底传递的是什么东西呢？这其实是程序设计中**求值策略（Evaluation strategies）**&#x7684;概念。在计算机科学中，求值策略是确定编程语言中表达式的求值的一组（通常确定性的）规则。求值策略定义何时和以何种顺序求值给函数的实际参数、什么时候把它们代换入函数、和代换以何种形式发生。

&#x20;       求值策略分为两大基本类，基于如何处理给函数的实际参数，分为严格的和非严格的。在“严格求值”中，函数调用过程中，给函数的实际参数总是在应用这个函数之前求值。在严格求值中有几个关键的求值策略是我们比较关心的，那就是**传值调用**（Call by value）、**传引用调用**（Call by reference）以及**传共享对象调用**（Call by sharing）。

* 传值调用（值传递）
  * 在传值调用中，实际参数先被求值，然后其值通过复制，被传递给被调函数的形式参数。因为形式参数拿到的只是一个"局部拷贝"，所以如果在被调函数中改变了形式参数的值，并不会改变实际参数的值。
* 传引用调用（引用传递）
  * 在传引用调用中，传递给函数的是它的实际参数的隐式引用而不是实参的拷贝。因为传递的是引用，所以，如果在被调函数中改变了形式参数的值，改变对于调用者来说是可见的。
* 传共享对象调用（共享对象传递）
  * 传共享对象调用中，先获取到实际参数的地址，然后将其复制，并把该地址的拷贝传递给被调函数的形式参数。因为参数的地址都指向同一个对象，所以我们也称之为"传共享对象"，所以，如果在被调函数中改变了形式参数的值，调用者是可以看到这种变化的。

&#x20;       **其实Java中使用的求值策略就是传共享对象调用，也就是说，Java会将对象的地址的拷贝传递给被调函数的形式参数**

### **Q8. 为什么不能用浮点型表示金额？**

&#x20;       由于计算机中保存的小数其实是十进制的小数的近似值，并不是准确值，所以，千万不要在代码中使用浮点数来表示金额等重要的指标。建议使用**BigDecimal**或者Long（单位为分）来表示金额。

### Q9. Integer的缓存机制

&#x20;       Integer缓存机制是在Java 5中引入的一个有助于节省内存、提高性能的功能。下面是Integer缓存的示例代码：

```
package com.javapapers.java;

public class JavaIntegerCache {
    public static void main(String... strings) {

        Integer integer1 = 3;
        Integer integer2 = 3;

        if (integer1 == integer2)
            System.out.println("integer1 == integer2");
        else
            System.out.println("integer1 != integer2");

        Integer integer3 = 300;
        Integer integer4 = 300;

        if (integer3 == integer4)
            System.out.println("integer3 == integer4");
        else
            System.out.println("integer3 != integer4");

    }
}
```

上面这段代码真正的输出结果：

```
integer1 == integer2
integer3 != integer4
```

&#x20;       在Java中，`==`比较的是对象应用，而`equals`比较的是值。所以，在这个例子中，不同的对象有不同的引用，所以正常情况下在进行比较的时候都将返回false。奇怪的是，这里两个类似的if条件判断返回不同的布尔值。造成这种现象的原因是：

&#x20;      **在Java 5中，在Integer的操作上引入了一个新功能来节省内存和提高性能。整型对象通过使用相同的对象引用实现了缓存和重用。**

> 适用于整数值区间-128 至 +127。
>
> 只适用于自动装箱。使用构造函数创建对象不适用。

&#x20;       Java的编译器把基本数据类型自动转换成封装类对象的过程叫做`自动装箱`，相当于使用`valueOf`方法，JDK中的`valueOf`方法的实现如下：

```
/**
     * Returns an {@code Integer} instance representing the specified
     * {@code int} value.  If a new {@code Integer} instance is not
     * required, this method should generally be used in preference to
     * the constructor {@link #Integer(int)}, as this method is likely
     * to yield significantly better space and time performance by
     * caching frequently requested values.
     *
     * This method will always cache values in the range -128 to 127,
     * inclusive, and may cache other values outside of this range.
     *
     * @param  i an {@code int} value.
     * @return an {@code Integer} instance representing {@code i}.
     * @since  1.5
     */
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

IntegerCache是Integer类中定义的一个`private static`的内部类。它的定义如下：

```
  /**
     * Cache to support the object identity semantics of autoboxing for values between
     * -128 and 127 (inclusive) as required by JLS.
     *
     * The cache is initialized on first usage.  The size of the cache
     * may be controlled by the {@code -XX:AutoBoxCacheMax=} option.
     * During VM initialization, java.lang.Integer.IntegerCache.high property
     * may be set and saved in the private system properties in the
     * sun.misc.VM class.
     */

    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }

```



### **Q10. 对象的创建过程**



### **Q11. 抽象类与接口**



### **Q12. 内部类**



















