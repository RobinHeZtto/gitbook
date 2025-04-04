# Java泛型

> **泛型程序设计**（generic programming）是程序设计语言的一种风格或范式。泛型允许程序员在[强类型程序设计语言](https://zh.wikipedia.org/wiki/%E5%BC%B7%E9%A1%9E%E5%9E%8B%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80)中编写代码时使用一些以后才指定的类型，在实例化时作为参数指明这些类型。

### 1. 什么是泛型       &#x20;

&#x20;        Java泛型（generic）是JDK 5中引入的一个新特性，其本质是**参数化类型**，也就是说所操作的数据类型被指定为一个参数（type parameter），这种参数类型可以用在**类**、**接口**和**方法**的创建中，分别称为**泛型类**、**泛型接口、泛型方法。**

> 类型参数与类型变量：Foo\<T>中的T为类型参数,  Foo\<String>中的String为类型变量

&#x20;       与非泛型代码相比，使用泛型的代码具有许多优点：&#x20;

*   **在编译时进行更强的类型检查**。Java编译器将强类型检查应用于通用代码，如果代码违反类型安全，则会发出错误。修复编译时错误比修复运行时错误容易，后者可能很难找到。&#x20;

    ```
    ArrayList arr = new ArrayList();
    arr.add("string");
    arr.add(100);
    for (int i = 0; i < arr.size(); i++) {
        String s1 = (String) arr.get(i);
    }

    运行时出现异常：
    java.lang.ClassCastException：java.lang.Integer cannot be cast to java.lang.String
    ```
*   **消除类型转换**。以下不带泛型的代码段需要强制转换：

    ```
    ArrayList arrayList = new ArrayList();
    arrayList.add("no Generic");
    // 没有使用泛型，需要类型转换
    String s = (String) arrayList.get(0);

    ArrayList<String> arrayList1 = new ArrayList<>();
    arrayList1.add("Generic");
    // 使用泛型，不需要类型转换
    String s1 = arrayList1.get(0);
    ```
* **使程序员能够实现通用算法，增强代码的复用性**。通过使用泛型，程序员可以实现对不同类型的集合进行工作，可以自 定义并且类型安全且易于阅读的泛型算法。

### 2. 泛型的使用

 泛型接口：

```
public interface GenericInterface<T> {
}
```

泛型类：

```
public class GenericClass<T, E> implements GenericInterface<E> {
}
```

泛型方法：

```
定义：
public <T, V> void genericMethod() {
}

调用：
Test.<T, V>genericMethod()
或
Test.genericMethod()
```

常见的类型变量名：

* E：Element（一般在集合中使用）&#x20;
* K：Key（键）&#x20;
* N：Number（数字类型）
* T：Type（类型）&#x20;
* V：Value（数字）
* S，U，V ：第二，第三，第四个类型

泛型原始类型：

### 3. 泛型通配符

泛型通配符主要针对以下两种需求：

* 从一个泛型集合里面读取元素
* 往一个泛型集合里面插入元素

#### **3.1 无限定通配符 ？**

非限定通配符 ？ 既不能读也不能写

#### **3.2 上界通配符（? extends）**

？extends  上界 可以读但不能写

#### **3.3 下界通配符(? super)**

？super 下界 能够写不能读

### 4. 泛型的本质

类型擦除：

```
ArrayList<String> strings = new ArrayList<>();
ArrayList<Integer> integers = new ArrayList<>();
System.out.println(strings.getClass() == integers.getClass());
```

如何擦除： 当擦除泛型类型后，留下的就只有原始类型了，例如上面的代码，原始类型就是ArrayList。擦除类型变量，并替换为限定类型（T为无限定的类型变量，用Object替换），如下所示

擦除之前：

```
//泛型类型  
class Pair<T> {    
    private T value;    
    public T getValue() {    
        return value;    
    }    
    public void setValue(T  value) {    
        this.value = value;    
    }    
}
```

擦除之后：

```
//原始类型  
class Pair {    
    private Object value;    
    public Object getValue() {    
        return value;    
    }    
    public void setValue(Object  value) {    
        this.value = value;    
    }    
}
```

因为在Pair\<T>中，T是一个无限定的类型变量，所以用Object替换。如果是Pair\<T extends Number>，擦除后，类型变量用Number类型替换。
