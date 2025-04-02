# 构造者模式

&#x20;       建造者模式（Builder Pattern）使用多个简单的对象一步步构建成一个复杂的对象，将一个复杂对象的**构建与其表示分离**，使得**同样的构建过程可以创建不同的表示**，这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

![经典Builder模式类图](<../../../.gitbook/assets/image (471).png>)

&#x20;       从上图可以看到，经典Buider模式中有四个角色，以组装电脑为例：

1. 要建造的产品Product -- 组装的电脑
2. 抽象的Builder -- 装CPU、内存条、硬盘等抽象的步骤
3. Builder的具体实现ConcreteBuilder -- 对上述抽象步骤的实现，比如装i5CPU、8G内存条、1T硬盘
4. 使用者Director -- 电脑装机人员

&#x20;       在代码实现中，一般不是按照经典的写法，大多按如下的写法：

```
public class Computer {
    private String cpu;
    private String ram;
    private String display;

    Computer(Builder builder) {
        this.cpu = builder.cpu;
        this.ram = builder.ram;
        this.display = builder.display;
    }

    public static class Builder{
        private String cpu = "Bloomfield";
        private String ram = "mem32";
        private String display;

        public Builder setCpu(String cpu) {
            this.cpu = cpu;
            return this;
        }

        public Builder setRam(String ram) {
            this.ram = ram;
            return this;
        }

        public Builder setDisplay(String display) {
            this.display = display;
            return this;
        }
        
        public Computer build() {
            return new Computer(this);
        }
    }
}
```
