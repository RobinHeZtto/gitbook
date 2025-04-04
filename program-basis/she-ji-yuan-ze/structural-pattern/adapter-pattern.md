# 适配器模式

### 1. 什么是适配器模式

&#x20;       **适配器模式**（Adapter Pattern）属于结构型模式的一种，是作为两个不兼容的接口之间的桥梁，将一个类的接口转换成客户希望的另外一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。直白点描述，适配器类似于粘合剂，将二个不兼容的类粘合在一起，使他们能够互相协助起来。

&#x20;       **使用场景：**

* 系统需要使用现有的类，而此类的接口不符合系统的需要。
* 想要建立一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作，这些源类不一定有一致的接口。&#x20;
* 通过接口转换，将一个类插入另一个类系中。（比如老虎和飞禽，现在多了一个飞虎，在不增加实体的需求下，增加一个适配器，在里面包容一个虎对象，实现飞的接口。）

### 2. 类适配器

&#x20;       如下图所示，目标接口需要的是operation2()，但是Adaptee对象只有operation3()，所以就出现了不兼容的情况，Adapter实现了一个operation2()将Adaptee的operation3()转换成Target所需要的operation2()。           类适配器通过实现Target接口及继承Adaptee类来实现接口转换。

![](<../../../.gitbook/assets/image (481).png>)

* Target：目标类，即期待得到的接口。
* Adaptee：需要适配的接口。
* Adapter：适配器，将源接口转换成目标接口，必须是具体类。

下面以生活电压220V转换成USB适配电压5V为例的demo演示类适配器模式。

```
/**
 * Target角色
 */
public interface FiveVolt {
    public int getVolt5();
}
```

```
/**
 * Adaptee角色，需要适配的对象
 */
public class Volt220 {
    public int getVolt220() {
        return 220;
    }
}
```

```
/**
 * Adapter角色，将220V电压转换成5V电压
 */
public class VoltAdapter extends Volt220 implements FiveVolt{

    @Override
    public int getVolt5() {
        return 5;
    }
} 
```

```
public class Client {
    public static void main(String[] args) {
        VoltAdapter adapter = new VoltAdapter();
        System.out.println("输出电压：" + adapter.getVolt5());
    }
}
```

### 2. 对象适配器

&#x20;       与类适配器不同的是，对象适配器不再使用继承关系连接到Adaptee类，而是使用代理关系连接到Adaptee类。

![](<../../../.gitbook/assets/image (439).png>)

```
public class VoltAdapter2 implements FiveVolt{
    
    Volt220 mVolt220;
    
    public VoltAdapter2(Volt220 volt220) {
        mVolt220 = volt220;
    }

    public int getVolt220() {
        return mVolt220.getVolt220();
    }
    
    @Override
    public int getVolt5() {
        return 5;
    }
}
```

```
public class Client {
    public static void main(String[] args) {
        VoltAdapter2 voltAdapter2 = new VoltAdapter2(new Volt220());
        System.out.println("输出电压：" + voltAdapter2.getVolt5());
    }
}
```
