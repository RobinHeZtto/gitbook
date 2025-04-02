# 享元模式

### 1.定义

享元模式（Flyweight Pattern）主要用于减少创建对象的数量，以减少内存占用和提高性能。这种类型的设计模式属于结构型模式，它提供了减少对象数量从而改善应用所需的对象结构的方式。

![](<../../../.gitbook/assets/image (29).png>)

**角色说明：**

* Flyweight(抽象享元角色)：接口或抽象类，可以同时定义出对象的外部状态和内部状态的接口或实现。
* ConcreteFlyweight（具体享元角色）：实现抽象享元角色中定义的业务。
* UnsharedConcreteFlyweight（不可共享的享元角色）：并不是所有的抽象享元类的子类都需要被共享，不能被共享的子类可设计为非共享具体享元类；当需要一个非共享具体享元类的对象时可以直接通过实例化创建。该对象一般不会出现在享元工厂中。
* FlyweightFactory（享元工厂）：管理对象池和创建享元对象。

### 2.使用场景

主要解决：在有大量对象时，有可能会造成内存溢出，我们把其中共同的部分抽象出来，如果有相同的业务请求，直接返回在内存中已有的对象，避免重新创建。

* 系统中有大量对象。
* 这些对象消耗大量内存。
* 这些对象的状态大部分可以外部化。

如何解决：用唯一标识码判断，如果在内存中有，则返回这个唯一标识码所标识的对象。\
关键代码：用 HashMap等数据结构存储这些对象。

### 3.使用示例

Android中的Message就使用到了亨元模式，messge使用完后放进池中，避免重复创建多余的message。

```
public final class Message implements Parcelable {
    public static final Object sPoolSync = new Object();
    // 定义message池
    private static Message sPool;
    private static int sPoolSize = 0;
    // 指向下一个message
    Message next;
    
    // 从池中获取message复用
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    // 重新将对象放进池中
    void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}
```

