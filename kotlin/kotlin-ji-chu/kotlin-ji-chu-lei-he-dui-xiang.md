# Kotlin基础-类和对象

## 属性

#### 声明属性

Kotlin 类中的属性既可以用关键字 _var_ 声明为可变的，也可以用关键字 _val_ 声明为只读的。

#### Getters 与 Setters

声明一个属性的完整语法是：

```
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```

其初始器（initializer）、getter 和 setter 都是可选的。属性类型必须初始化，可以从初始器或者从其 getter 返回值。

一个只读属性的语法和一个可变的属性的语法有两方面的不同：

* 只读属性的用 `val`开始代替`var`&#x20;
* 只读属性不允许 setter

#### 延迟初始化属性与变量



## 扩展：



## 委托： <a href="#wei-tuo" id="wei-tuo"></a>

#### 延迟属性 Lazy

lazy() 是一个函数, 接受一个 Lambda 表达式作为参数, 返回一个 Lazy \<T> 实例的函数，返回的实例可以作为实现延迟属性的委托： 第一次调用 get() 会执行已传递给 lazy() 的 lamda 表达式并记录结果， 后续调用 get() 只是返回记录的结果。









@JvmField



## @JvmOverloads



by noOpDelegate()



```
@get:JvmName("dispatcher")
```
