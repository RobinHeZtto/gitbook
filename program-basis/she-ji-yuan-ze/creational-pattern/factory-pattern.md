# 工厂模式

工厂模式（Factory Pattern）是 最常用的设计模式之一，属于创建型模式的一种。在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

根据产品是具体产品还是具体工厂可分为**简单工厂模式**和**工厂方法模式**，根据工厂的抽象程度可分为**工厂方法模式**和**抽象工厂模式**

### 简单工厂模式

该模式对对象创建管理方式最为简单，因为其仅仅简单的对不同类对象的创建进行了一层薄薄的封装。该模式通过向工厂传递类型来指定要创建的对象，其UML类图如下：

![](https://img2018.cnblogs.com/blog/1419489/201906/1419489-20190628144601084-563759643.png)

下面我们使用手机生产来讲解该模式：

**Phone类**：手机标准规范类(AbstractProduct)

```
public interface Phone {
    void make();
}
```

&#x20;**MiPhone类**：制造小米手机（Product&#x31;**）**

```
public class MiPhone implements Phone {
    public MiPhone() {
        this.make();
    }
    @Override
    public void make() {
        // TODO Auto-generated method stub
        System.out.println("make xiaomi phone!");
    }

```

**IPhone类**：制造苹果手机（Product&#x32;**）**

```
public class IPhone implements Phone {
    public IPhone() {
        this.make();
    }
    @Override
    public void make() {
        // TODO Auto-generated method stub
        System.out.println("make iphone!");
    }

```

**PhoneFactory类**：手机代工厂（Factory）

```
public class PhoneFactory {
    public Phone makePhone(String phoneType) {
        if(phoneType.equalsIgnoreCase("MiPhone")){
            return new MiPhone();
        }
        else if(phoneType.equalsIgnoreCase("iPhone")) {
            return new IPhone();
        }
        return null;
    }
｝
```

&#x20;**演示**

```
public class Demo {
    public static void main(String[] arg) {
        PhoneFactory factory = new PhoneFactory();
        Phone miPhone = factory.makePhone("MiPhone");            // make xiaomi phone!
        IPhone iPhone = (IPhone)factory.makePhone("iPhone");    // make iphone!
    }

```

### 工厂方法模式(Factory Method)

和简单工厂模式中工厂负责生产所有产品相比，工厂方法模式将生成具体产品的任务分发给具体的产品工厂，其UML类图如下：

![](https://img2018.cnblogs.com/blog/1419489/201906/1419489-20190628154133368-906051111.png)

也就是定义一个抽象工厂，其定义了产品的生产接口，但不负责具体的产品，将生产任务交给不同的派生类工厂。这样不用通过指定类型来创建对象了。继续使用生产手机的例子来讲解该模式。其中和产品相关的Phone类、MiPhone类和IPhone类的定义不变。

**AbstractFactory类**：生产不同产品的工厂的抽象类

```
public interface AbstractFactory {
    Phone makePhone();
}
```

**XiaoMiFactory类**：生产小米手机的工厂（ConcreteFactory1）

```
public class XiaoMiFactory implements AbstractFactory{
    @Override
    public Phone makePhone() {
        return new MiPhone();
    }
}
```

**AppleFactory类**：生产苹果手机的工厂（ConcreteFactory2）

```
public class AppleFactory implements AbstractFactory {
    @Override
    public Phone makePhone() {
        return new IPhone();
    }
}
```

**演示**

```
public class Demo {
    public static void main(String[] arg) {
        AbstractFactory miFactory = new XiaoMiFactory();
        AbstractFactory appleFactory = new AppleFactory();
        miFactory.makePhone();            // make xiaomi phone!
        appleFactory.makePhone();        // make iphone!
    }

```

### 抽象工厂模式(Abstract Factory)

上面两种模式不管工厂怎么拆分抽象，都只是针对一类产品**Phone**（AbstractProduct），如果要生成另一种产品　PC，应该怎么表示呢？

最简单的方式是把2中介绍的工厂方法模式完全复制一份，不过这次生产的是PC。但同时也就意味着我们要完全复制和修改Phone生产管理的所有代码，显然这是一个笨办法，并不利于扩展和维护。

抽象工厂模式通过在AbstarctFactory中增加创建产品的接口，并在具体子工厂中实现新加产品的创建，当然前提是子工厂支持生产该产品。否则继承的这个接口可以什么也不干。

其UML类图如下：

![](https://img2018.cnblogs.com/blog/1419489/201906/1419489-20190628170705865-1781414242.png)

从上面类图结构中可以清楚的看到如何在工厂方法模式中通过增加新产品接口来实现产品的增加的。继续通过小米和苹果产品生产的例子来解释该模式。为了弄清楚上面的结构，我们使用具体的产品和工厂来表示上面的UML类图，能更加清晰的看出模式是如何演变的：

![](https://img2018.cnblogs.com/blog/1419489/201906/1419489-20190628164001258-637961514.png)

**PC类**：定义PC产品的接口(AbstractPC)

```
public interface PC {
    void make();
}
```

**MiPC类**：定义小米电脑产品(MIPC

```
public class MiPC implements PC {
    public MiPC() {
        this.make();
    }
    @Override
    public void make() {
        // TODO Auto-generated method stub
        System.out.println("make xiaomi PC!");
    }

```

**MAC类**：定义苹果电脑产品(MAC

```
public class MAC implements PC {
    public MAC() {
        this.make();
    }
    @Override
    public void make() {
        // TODO Auto-generated method stub
        System.out.println("make MAC!");
    }

```

下面需要修改工厂相关的类的定义：

**AbstractFactory类**：增加PC产品制造接口

```
public interface AbstractFactory {
    Phone makePhone();
    PC makePC();
}
```

**XiaoMiFactory类**：增加小米PC的制造（ConcreteFactory1）

```
public class XiaoMiFactory implements AbstractFactory{
    @Override
    public Phone makePhone() {
        return new MiPhone();
    }
    @Override
    public PC makePC() {
        return new MiPC();
    }

```

**AppleFactory类**：增加苹果PC的制造（ConcreteFactory2）

```
public class AppleFactory implements AbstractFactory {
    @Override
    public Phone makePhone() {
        return new IPhone();
    }
    @Override
    public PC makePC() {
        return new MAC();
    }

```

**演示**

```
public class Demo {
    public static void main(String[] arg) {
        AbstractFactory miFactory = new XiaoMiFactory();
        AbstractFactory appleFactory = new AppleFactory();
        miFactory.makePhone();            // make xiaomi phone!
        miFactory.makePC();                // make xiaomi PC!
        appleFactory.makePhone();        // make iphone!
        appleFactory.makePC();            // make MAC!
    }

```

### 总结

上面介绍的三种工厂模式有各自的应用场景，实际应用时能解决问题满足需求即可，可灵活变通，无所谓高级与低级。外无论哪种模式，由于可能封装了大量对象和工厂创建，新加产品需要修改已定义好的工厂相关的类，因此对于产品和工厂的扩展不太友好，利弊需要权衡一下。&#x20;
