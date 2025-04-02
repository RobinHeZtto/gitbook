# Kotlin基础-构造函数

Kotlin的构造函数分为**主构造器（primary constructor）**&#x548C;**次级构造器（secondary constructor）**。下面我们来看看他们的写法。

### 1. Primary Constructor

**写法一：**

class 类名 constructor(形参1, 形参2, 形参3){}

```
class Person constructor(username: String, age: Int){
	private val username: String
	private var age: Int

	init{
		this.username = username
		this.age = age
	}
}
```

这里需要注意几点：

* 关键字**constructor**：在Java中，构造方法名须和类名相同；而在Kotlin中，是通过constructor关键字来标明的，且**对于Primary Constructor而言，它的位置是在类的首部（class header）而不是在类体中（class body）**。
* 关键字init：**init{}它被称作是初始化代码块（Initializer Block），它的作用是为了Primary Constructor服务的**，由于Primary Constructor是放置在类的首部，是不能包含任何初始化执行语句的，这是语法规定的，那么这个时候就有了init的用武之地，我们可以把初始化执行语句放置在此处，为属性进行赋值。

**写法二（演变一）：**

当constructor关键字没有注解和可见性修饰符作用于它时，constructor关键字可以省略（当然，如果有这些修饰时，是不能够省略的，并且constructor关键字位于修饰符后面）。那么上面的代码就变成：

```
class Person (username: String, age: Int){
    private val username: String
    private var age: Int

    init{
	    this.username = username
	    this.age = age
    }
}
```

初始化执行语句不是必须放置在init块中，我们可以在定义属性时直接将主构造器中的形参赋值给它。

```
class Person(username: String, age: Int){
    private val username: String = username
    private var age: Int = age
}
```

可以看出，我们的写法二实际上就是对我们在写法一前面提到的两个关键字的简化。

_写法三（演变二）：_\
这种在构造器中声明形参，然后在属性定义进行赋值，这个过程实际上很繁琐，有没有更加简便的方法呢？当然有，我们可以直接在Primary Constructor中定义类的属性。

```
class Person(private val username: String, private var age: Int){}
```

如果类不包含其他操作函数，那么连花括号也可以省略

```
class Person(private val username: String, private var age: Int)
```

看，是不是一次比一次简洁？实际上这就是Kotlin的一大特点

4\.    当我们定义一个类时，我们如果没有为其显式提供Primary Constructor，Kotlin编译器会默认为其生成一个无参主构造，这点和Java是一样的。比如有这样的一个类：

```
class Person {
    private val username = "David"
    private var age = 23

    fun printInfo() {
        println("username = $username, age = $age")
    }
}
fun main(args: Array<String>) {
    val person = Person()
    person.printInfo()
}
```

我们使用javap命令来反编译这个Person类。可以看到这个无参主构造：

![](<../../.gitbook/assets/image (143).png>)

二、 Secondary Constructor\
1.示例：\


```
class User{
	private val username: String
	private var age: Int
		
	constructor(username: String, age: Int){
	    this.username = username
	    this.age = age
	}
}复制代码
```

和Primary Constructor相比，很明显的一点，Secondary Constructor是定义在类体中。第二，Secondary Constructor可以有多个，而Primary Constructor只会有一个。2. 要想实现属性的初始化，实际上主构造器已经能够应付多数情况了，为什么还需要次级构造器？主要原因是因为我们有时候是需要去继承框架中的类。如在Android中你自定义一个Button：\


```
class MyButton : AppCompatButton {

    constructor(context: Context) : this(context, null)
    constructor(context: Context, attrs: AttributeSet?) : this(context, attrs, R.attr.buttonStyle)
    constructor(context: Context, attrs: AttributeSet?, defStyleAttr: Int) : super(context, attrs, defStyleAttr)
}复制代码
```

在这种情况下，你如果需要重写AppCompatButton的多个构造器，那么，就需要用到Secondary Constructor。同样，这里也有几点需要注意：

* 可以看到，我们可以使用this关键字来调用自己的其他构造器，并且需要注意它的语法形式，次级构造器: this(参数列表)
* 可以使用super关键字来调用父类构造器，当然这块内容我们放到继承那块再来介绍。

3\. 我们再来看这样一种情况，我们同时定义了主构造器和次级构造器：\


```
class Student constructor(username: String, age: Int) {


    private val username: String = username
    private var age: Int = age
    private var address: String
    private var isMarried: Boolean


    init {
        this.address = "Beijing"
        this.isMarried = false
    }


    constructor(username: String, age: Int, address: String) :this(username, age) {

        this.address = address
    }


    constructor(username: String, age: Int, address: String, isMarried: Boolean) : this(username, age, address) {

        this.isMarried = isMarried
    }

}
```

可以看到，四个参数的次级构造调用三个参数的次级构造，而三个参数的次级构造又调用了主构造。换句话，次级构造会直接或者间接调用主构造。这也就是这个例子需要说明的问题。如果你还不信，我们可以把“: this(username, age)”删除掉，看编译器会给你什么提示：\
<img src="https://user-gold-cdn.xitu.io/2018/10/10/1665d87fe3b3b1d3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="" data-size="original">\
“Primary constructor call expected”。提示很明显，和我们在之前提到的内容是一样的。
