# 访问者模式

访问者模式是一种将**算法与对象结构分离**的**行为型设计模式**。访问者模式基本思想如下：

首先我们拥有一个由许多对象构成的对象结构，这些对象的类都拥有一个accept方法用来接受访问者对象；访问者是一个接口，它拥有一个visit方法，这个方法对访问到的对象结构中不同类型的元素作出不同的反应；在对象结构的一次访问过程中，我们遍历整个对象结构，对每一个元素都实施accept方法，在每一个元素的accept方法中回调访问者的visit方法，从而使访问者得以处理对象结构的每一个元素。我们可以针对对象结构设计不同的实在的访问者类来完成不同的操作。

### 访问者模式的角色 <a href="#usercontent-fang-wen-zhe-mo-shi-de-jie-gou" id="usercontent-fang-wen-zhe-mo-shi-de-jie-gou"></a>

访问者模式结构中包含以下5个角色：

* Visitor（抽象访问者）：抽象访问者为对象结构中每一个具体元素类ConcreteElement声明一个访问操作，从这个操作的名称或参数类型可以清楚知道需要访问的具体元素的类型，具体访问者则需要实现这些操作方法，定义对这些元素的访问操作。
* ConcreteVisitor（具体访问者）：具体访问者实现了抽象访问者声明的方法，每一个操作作用于访问对象结构中一种类型的元素。
* Element（抽象元素）：一般是一个抽象类或接口，定义一个Accept方法，该方法通常以一个抽象访问者作为参数。
* ConcreteElement（具体元素）：具体元素实现了Accept方法，在Accept方法中调用访问者的访问方法以便完成一个元素的操作。
* ObjectStructure（对象结构）：对象结构是一个元素的集合，用于存放元素对象，且提供便利其内部元素的方法。

![](<../../../.gitbook/assets/image (60).png>)

### 访问者模式的实现 <a href="#usercontent-fang-wen-zhe-mo-shi-de-shi-xian" id="usercontent-fang-wen-zhe-mo-shi-de-shi-xian"></a>

汽车作为对象结构的角色，里面包含引擎，车身等部分对象，访问者角色对象为PrintVisitor，车接受该访问者让其访问车的各个组成对象并打印信息；

#### visitor <a href="#user-content-visitor" id="user-content-visitor"></a>

访问者接口，包含三个方法

```
public interface Visitor {
    void visit(Engine engine);
    void visit(Body body);
    void visit(Car car);
}
```

#### 具体访问者 <a href="#usercontent-ju-ti-fang-wen-zhe" id="usercontent-ju-ti-fang-wen-zhe"></a>

汽车打印访问者：

```
public class PrintCar implements Visitor {
    public void visit(Engine engine) {
        System.out.println("Visiting engine");
    }
    public void visit(Body body) {
        System.out.println("Visiting body");
    }
    public void visit(Car car) {
        System.out.println("Visiting car");
    }
}
```

汽车检修的访问者：

```
public class CheckCar implements Visitor {
    public void visit(Engine engine) {
        System.out.println("Check engine");
    }
    public void visit(Body body) {
        System.out.println("Check body");
    }
    public void visit(Car car) {
        System.out.println("Check car");
    }
}
```

#### 抽象元素 <a href="#usercontent-chou-xiang-yuan-su" id="usercontent-chou-xiang-yuan-su"></a>

```
public interface Visitable {
    void accept(Visitor visitor);
}
```

#### 具体元素 <a href="#usercontent-ju-ti-yuan-su" id="usercontent-ju-ti-yuan-su"></a>

```
public class Body implements Visitable {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

```
public class Engine implements Visitable {
    @Override    
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

#### 对象结构 <a href="#usercontent-dui-xiang-jie-gou" id="usercontent-dui-xiang-jie-gou"></a>

```
public class Car {
    private List<Visitable> visit = new ArrayList<>();
    public void addVisit(Visitable visitable) {
        visit.add(visitable);
    }
    public void show(Visitor visitor) {
        for (Visitable visitable: visit) {
            visitable.accept(visitor);
        }
    }
}
```

当前访问者模式例子中的对象结构，这个列表是元素（Visitable）的集合，这便是对象结构的通常表示，它一般会是一堆元素的集合，不过这个集合不一定是列表，也可能是树，链表等等任何数据结构，甚至是若干个数据结构。其中show方法，就是汽车类的精髓，它会枚举每一个元素，让访问者访问。

#### 客户端 <a href="#usercontent-ke-hu-duan" id="usercontent-ke-hu-duan"></a>

```
public class Client {
    static public void main(String[] args) {
        Car car = new Car();
        car.addVisit(new Body());
        car.addVisit(new Engine());
        
        Visitor print = new PrintCar();
        car.show(print);
    }
}
```

汽车中的元素很稳定，这些几乎不可能改变，而最容易改变的就是访问者这部分。**访问者模式最大的优点就是增加访问者非常容易**，我们从代码上来看，如果要增加一个访问者，你只需要做一件事即可，新增访问者实现Visitor接口，然后就可以直接调用对象结构的show方法去访问汽车了。

### 访问者模式使用场景

1，字节码操作框架ASM

2，html解析库Jsoup

### 总结 <a href="#usercontent-zong-jie" id="usercontent-zong-jie"></a>

* 增加新的元素类很困难，需要在每一个访问者类中增加相应访问操作代码，这违背了开闭原则；
* 元素对象有时候必须暴露一些自己的内部操作和状态，否则无法供访问者访问，这破坏了元素的封装性。

优点：

* 增加新的访问操作十分方便，符合开闭原则；
* 将有关元素对象的访问行为集中到一个访问者对象中，而不是分散在一个个的元素类中，类的职责更加清晰，符合单一职责原则

缺点：

* 增加新的元素类很困难，需要在每一个访问者类中增加相应访问操作代码，这违背了开闭原则；
* 元素对象有时候必须暴露一些自己的内部操作和状态，否则无法供访问者访问，这破坏了元素的封装性。

### &#x20;<a href="#usercontent-zong-jie" id="usercontent-zong-jie"></a>

\


