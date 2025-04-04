# JVM-垃圾回收

### 1. 判断一个对象是否可被回收 <a href="#pan-duan-yi-ge-dui-xiang-shi-fou-ke-bei-hui-shou" id="pan-duan-yi-ge-dui-xiang-shi-fou-ke-bei-hui-shou"></a>

#### 1.1 引用计数算法 <a href="#id-1-yin-yong-ji-shu-suan-fa" id="id-1-yin-yong-ji-shu-suan-fa"></a>

给对象添加一个引用计数器，当对象增加一个引用时计数器加 1，引用失效时计数器减 1。引用计数为 0 的对象可被回收。

两个对象出现循环引用的情况下，此时引用计数器永远不为 0，导致无法对它们进行回收。**正因为循环引用的存在，因此 Java 虚拟机不使用引用计数算法。**

```
public class ReferenceCountingGC {

    public Object instance = null;

    public static void main(String[] args) {
        ReferenceCountingGC objectA = new ReferenceCountingGC();
        ReferenceCountingGC objectB = new ReferenceCountingGC();
        objectA.instance = objectB;
        objectB.instance = objectA;
    }
}

```

#### 1.2 可达性分析算法 <a href="#id-2-ke-da-xing-fen-xi-suan-fa" id="id-2-ke-da-xing-fen-xi-suan-fa"></a>

通过 GC Roots 作为起始点进行搜索，能够到达到的对象都是存活的，不可达的对象可被回收。

![](<../.gitbook/assets/image (289).png>)

Java 虚拟机使用该算法来判断对象是否可被回收，在 Java 中 GC Roots 一般包含以下内容:

* 虚拟机栈中引用的对象
* 本地方法栈中引用的对象
* 方法区中类静态属性引用的对象
* 方法区中的常量引用的对象

#### 1.3 方法区的回收 <a href="#id-3-fang-fa-qu-de-hui-shou" id="id-3-fang-fa-qu-de-hui-shou"></a>

因为方法区主要存放永久代对象，而永久代对象的回收率比新生代低很多，因此在方法区上进行回收性价比不高。

主要是对常量池的回收和对类的卸载。在大量使用反射、动态代理、CGLib 等 ByteCode 框架、动态生成 JSP 以及 OSGi 这类频繁自定义 ClassLoader 的场景都需要虚拟机具备类卸载功能，以保证不会出现内存溢出。

类的卸载条件很多，需要满足以下三个条件，并且**满足了也不一定会被卸载:**

* 该类所有的实例都已经被回收，也就是堆中不存在该类的任何实例。
* **加载该类的 ClassLoader 已经被回收。**
* 该类对应的 Class 对象没有在任何地方被引用，也就无法在任何地方通过反射访问该类方法。

可以通过 -Xnoclassgc 参数来控制是否对类进行卸载。

#### 1.4 finalize() <a href="#id-4-finalize" id="id-4-finalize"></a>

finalize() 类似 C++ 的析构函数，用来做关闭外部资源等工作。但是 try-finally 等方式可以做的更好，并且该方法运行代价高昂，不确定性大，无法保证各个对象的调用顺序，因此最好不要使用。

当一个对象可被回收时，如果需要执行该对象的 finalize() 方法，那么就有可能通过在该方法中让对象重新被引用，从而实现自救。自救只能进行一次，如果回收的对象之前调用了 finalize() 方法自救，后面回收时不会调用 finalize() 方法。

### 2. 引用类型 <a href="#yin-yong-lei-xing" id="yin-yong-lei-xing"></a>

无论是通过引用计算算法判断对象的引用数量，还是通过可达性分析算法判断对象是否可达，判定对象是否可被回收都与引用有关。Java 具有四种强度不同的引用类型。

![](<../.gitbook/assets/image (468).png>)

#### 2.1 强引用 <a href="#id-1-qiang-yin-yong" id="id-1-qiang-yin-yong"></a>

被强引用关联的对象不会被回收。使用 new 一个新对象的方式来创建强引用。

```
Object obj = new Object();
```

#### 2.2 软引用 <a href="#id-2-ruan-yin-yong" id="id-2-ruan-yin-yong"></a>

被软引用关联的对象只有在内存不够的情况下才会被回收。使用 `SoftReference` 类来创建软引用。

```
Object obj = new Object();
SoftReference<Object> sf = new SoftReference<Object>(obj);
obj = null;  // 使对象只被软引用关联
```

#### 2.3 弱引用 <a href="#id-3-ruo-yin-yong" id="id-3-ruo-yin-yong"></a>

被弱引用关联的对象一定会被回收，也就是说它只能存活到下一次垃圾回收发生之前。使用 `WeakReference` 类来实现弱引用。

```
Object obj = new Object();
WeakReference<Object> wf = new WeakReference<Object>(obj);
obj = null;
```

#### 2.4 虚引用 <a href="#id-4-xu-yin-yong" id="id-4-xu-yin-yong"></a>

又称为**幽灵引用**或者**幻影引用**。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用取得一个对象。为一个对象设置虚引用关联的唯一目的就是能在这个对象被回收时收到一个系统通知。使用 `PhantomReference` 来实现虚引用。

```
Object obj = new Object();
PhantomReference<Object> pf = new PhantomReference<Object>(obj);
obj = null;
```

### 3. 垃圾回收算法 <a href="#la-ji-hui-shou-suan-fa" id="la-ji-hui-shou-suan-fa"></a>

#### 3.1 标记 - 清除 <a href="#id-1-biao-ji-qing-chu" id="id-1-biao-ji-qing-chu"></a>

最基础的垃圾回收算法，之所以说它是最基础的是因为它最容易实现，思想也是最简单的。标记-清除算法分为两个阶段：标记阶段和清除阶段。**标记阶段的任务是标记出所有需要被回收的对象，清除阶段就是回收被标记的对象所占用的空间**。具体过程如下图所示：&#x20;

![](<../.gitbook/assets/image (435).png>)

将存活的对象进行标记，然后清理掉未被标记的对象。

**不足:**

* 标记和清除过程效率都不高；
* 会产生大量不连续的内存碎片，导致无法给大对象分配内存。

#### 3.2 标记 - 整理 <a href="#id-2-biao-ji-zheng-li" id="id-2-biao-ji-zheng-li"></a>

![](<../.gitbook/assets/image (445).png>)

该算法标记阶段和标记-清除一样，但是在完成标记之后，它不是直接清理可回收对象，而是将存活对象都向一端移动，然后清理掉端边界以外的内存。

#### 3.3 复制 <a href="#id-3-fu-zhi" id="id-3-fu-zhi"></a>

![](<../.gitbook/assets/image (176).png>)

将内存划分为大小相等的两块，每次只使用其中一块，当这一块内存用完了就将还存活的对象复制到另一块上面，然后再把使用过的内存空间进行一次清理。

主要不足是只使用了内存的一半。另外复制算法的效率跟存活对象的数目多少有很大的关系，如果存活对象很多，那么Copying算法的效率将会大大降低。

现在的商业虚拟机都采用这种收集算法来回收新生代，但是并不是将新生代划分为大小相等的两块，而是分为一块较大的 Eden 空间和两块较小的 Survivor 空间，每次使用 Eden 空间和其中一块 Survivor。在回收时，将 Eden 和 Survivor 中还存活着的对象一次性复制到另一块 Survivor 空间上，最后清理 Eden 和使用过的那一块 Survivor。

HotSpot 虚拟机的 Eden 和 Survivor 的大小比例默认为 8:1，保证了内存的利用率达到 90%。如果每次回收有多于 10% 的对象存活，那么一块 Survivor 空间就不够用了，此时需要依赖于老年代进行分配担保，也就是借用老年代的空间存储放不下的对象。

#### 3.4 分代收集 <a href="#id-4-fen-dai-shou-ji" id="id-4-fen-dai-shou-ji"></a>

现在的商业虚拟机采用分代收集算法，它根据对象存活周期将内存划分为几块，不同块采用适当的收集算法。

一般将堆分为新生代和老年代。

* **新生代使用: 复制算法**
* **老年代使用: 标记 - 清除 或者 标记 - 整理 算法**

### 4. 垃圾收集器 <a href="#la-ji-shou-ji-qi" id="la-ji-shou-ji-qi"></a>

![](<../.gitbook/assets/image (147).png>)

以上是 HotSpot 虚拟机中的 7 个垃圾收集器，连线表示垃圾收集器可以配合使用。

* 单线程与多线程: 单线程指的是垃圾收集器只使用一个线程进行收集，而多线程使用多个线程；
* 串行与并行: 串行指的是垃圾收集器与用户程序交替执行，这意味着在执行垃圾收集的时候需要停顿用户程序；并形指的是垃圾收集器和用户程序同时执行。**除了 CMS 和 G1 之外，其它垃圾收集器都是以串行的方式执行。**

#### 4.1 Serial 收集器 <a href="#id-1serial-shou-ji-qi" id="id-1serial-shou-ji-qi"></a>

![](<../.gitbook/assets/image (283).png>)

Serial 翻译为串行，也就是说它以串行的方式执行。它是单线程的收集器，只会使用一个线程进行垃圾收集工作。

它的优点是简单高效，对于单个 CPU 环境来说，由于没有线程交互的开销，因此拥有最高的单线程收集效率。

它是 Client 模式下的默认新生代收集器，因为在用户的桌面应用场景下，分配给虚拟机管理的内存一般来说不会很大。Serial 收集器收集几十兆甚至一两百兆的新生代停顿时间可以控制在一百多毫秒以内，只要不是太频繁，这点停顿是可以接受的。

#### 4.2 ParNew 收集器 <a href="#id-2parnew-shou-ji-qi" id="id-2parnew-shou-ji-qi"></a>

![](<../.gitbook/assets/image (282).png>)

它是 Serial 收集器的多线程版本。是 Server 模式下的虚拟机首选新生代收集器，除了性能原因外，主要是因为除了 Serial 收集器，只有它能与 CMS 收集器配合工作。

默认开启的线程数量与 CPU 数量相同，可以使用 -XX:ParallelGCThreads 参数来设置线程数。

#### 4.3 Parallel Scavenge 收集器 <a href="#id-3parallelscavenge-shou-ji-qi" id="id-3parallelscavenge-shou-ji-qi"></a>

与 ParNew 一样是多线程收集器。其它收集器关注点是尽可能缩短垃圾收集时用户线程的停顿时间，而它的目标是达到一个可控制的吞吐量，它被称&#x4E3A;**“吞吐量优先”**&#x6536;集器。**这里的吞吐量指 CPU 用于运行用户代码的时间占总时间的比值。**

停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。

缩短停顿时间是以牺牲吞吐量和新生代空间来换取的: 新生代空间变小，垃圾回收变得频繁，导致吞吐量下降。

可以通过一个开关参数打卡 GC 自适应的调节策略(GC Ergonomics)，就不需要手工指定新生代的大小(-Xmn)、Eden 和 Survivor 区的比例、晋升老年代对象年龄等细节参数了。虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种方式称为 。

#### 4.4 Serial Old 收集器 <a href="#id-4serialold-shou-ji-qi" id="id-4serialold-shou-ji-qi"></a>

![](<../.gitbook/assets/image (179).png>)

是 Serial 收集器的老年代版本，也是给 Client 模式下的虚拟机使用。如果用在 Server 模式下，它有两大用途:

* 在 JDK 1.5 以及之前版本(Parallel Old 诞生以前)中与 Parallel Scavenge 收集器搭配使用。
* 作为 CMS 收集器的后备预案，在并发收集发生 Concurrent Mode Failure 时使用。

#### 4.5 Parallel Old 收集器 <a href="#id-5parallelold-shou-ji-qi" id="id-5parallelold-shou-ji-qi"></a>

![](<../.gitbook/assets/image (197).png>)

是 Parallel Scavenge 收集器的老年代版本。在注重吞吐量以及 CPU 资源敏感的场合，都可以优先考虑 Parallel Scavenge 加 Parallel Old 收集器。

#### 4.6 CMS 收集器 <a href="#id-6cms-shou-ji-qi" id="id-6cms-shou-ji-qi"></a>

![](<../.gitbook/assets/image (280).png>)

CMS(Concurrent Mark Sweep)，Mark Sweep 指的是标记 - 清除算法。

分为以下四个流程:

* 初始标记: 仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快，需要停顿。
* 并发标记: 进行 GC Roots Tracing 的过程，它在整个回收过程中耗时最长，不需要停顿。
* 重新标记: 为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要停顿。
* 并发清除: 不需要停顿。

在整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，不需要进行停顿。

具有以下缺点:

* 吞吐量低: 低停顿时间是以牺牲吞吐量为代价的，导致 CPU 利用率不够高。
* 无法处理浮动垃圾，可能出现 Concurrent Mode Failure。浮动垃圾是指并发清除阶段由于用户线程继续运行而产生的垃圾，这部分垃圾只能到下一次 GC 时才能进行回收。由于浮动垃圾的存在，因此需要预留出一部分内存，意味着 CMS 收集不能像其它收集器那样等待老年代快满的时候再回收。如果预留的内存不够存放浮动垃圾，就会出现 Concurrent Mode Failure，这时虚拟机将临时启用 Serial Old 来替代 CMS。
* 标记 - 清除算法导致的空间碎片，往往出现老年代空间剩余，但无法找到足够大连续空间来分配当前对象，不得不提前触发一次 Full GC。

#### 4.7 G1 收集器 <a href="#id-7g1-shou-ji-qi" id="id-7g1-shou-ji-qi"></a>

G1(Garbage-First)，它是一款面向服务端应用的垃圾收集器，在多 CPU 和大内存的场景下有很好的性能。HotSpot 开发团队赋予它的使命是未来可以替换掉 CMS 收集器。

堆被分为新生代和老年代，其它收集器进行收集的范围都是整个新生代或者老年代，而 G1 可以直接对新生代和老年代一起回收。

![](<../.gitbook/assets/image (112).png>)

G1 把堆划分成多个大小相等的独立区域(Region)，新生代和老年代不再物理隔离。

![](<../.gitbook/assets/image (267).png>)

通过引入 Region 的概念，从而将原来的一整块内存空间划分成多个的小空间，使得每个小空间可以单独进行垃圾回收。这种划分方法带来了很大的灵活性，使得可预测的停顿时间模型成为可能。通过记录每个 Region 垃圾回收时间以及回收所获得的空间(这两个值是通过过去回收的经验获得)，并维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的 Region。

每个 Region 都有一个 Remembered Set，用来记录该 Region 对象的引用对象所在的 Region。通过使用 Remembered Set，在做可达性分析的时候就可以避免全堆扫描。

![](<../.gitbook/assets/image (417).png>)

如果不计算维护 Remembered Set 的操作，G1 收集器的运作大致可划分为以下几个步骤:

* 初始标记
* 并发标记
* 最终标记: 为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程的 Remembered Set Logs 里面，最终标记阶段需要把 Remembered Set Logs 的数据合并到 Remembered Set 中。这阶段需要停顿线程，但是可并行执行。
* 筛选回收: 首先对各个 Region 中的回收价值和成本进行排序，根据用户所期望的 GC 停顿时间来制定回收计划。此阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分 Region，时间是用户可控制的，而且停顿用户线程将大幅度提高收集效率。

具备如下特点:

* 空间整合: 整体来看是基于“标记 - 整理”算法实现的收集器，从局部(两个 Region 之间)上来看是基于“复制”算法实现的，这意味着运行期间不会产生内存空间碎片。
* 可预测的停顿: 能让使用者明确指定在一个长度为 M 毫秒的时间片段内，消耗在 GC 上的时间不得超过 N 毫秒。

更详细内容请参考: [Getting Started with the G1 Garbage Collector (opens new window)](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)

### 5. 内存分配与回收策略 <a href="#nei-cun-fen-pei-yu-hui-shou-ce-lve" id="nei-cun-fen-pei-yu-hui-shou-ce-lve"></a>

#### 5.1 Minor GC 和 Full GC <a href="#minorgc-he-fullgc" id="minorgc-he-fullgc"></a>

* Minor GC: 发生在新生代上，因为新生代对象存活时间很短，因此 Minor GC 会频繁执行，执行的速度一般也会比较快。
* Full GC: 发生在老年代上，老年代对象其存活时间长，因此 Full GC 很少执行，执行速度会比 Minor GC 慢很多。

#### 5.2 内存分配策略 <a href="#nei-cun-fen-pei-ce-lve" id="nei-cun-fen-pei-ce-lve"></a>

**1. 对象优先在 Eden 分配**

大多数情况下，对象在新生代 Eden 区分配，当 Eden 区空间不够时，发起 Minor GC。

**2. 大对象直接进入老年代**

大对象是指需要连续内存空间的对象，最典型的大对象是那种很长的字符串以及数组。

经常出现大对象会提前触发垃圾收集以获取足够的连续空间分配给大对象。

-XX:PretenureSizeThreshold，大于此值的对象直接在老年代分配，避免在 Eden 区和 Survivor 区之间的大量内存复制。

**3. 长期存活的对象进入老年代**

为对象定义年龄计数器，对象在 Eden 出生并经过 Minor GC 依然存活，将移动到 Survivor 中，年龄就增加 1 岁，增加到一定年龄则移动到老年代中。

-XX:MaxTenuringThreshold 用来定义年龄的阈值。

**4. 动态对象年龄判定**

虚拟机并不是永远地要求对象的年龄必须达到 MaxTenuringThreshold 才能晋升老年代，如果在 Survivor 中相同年龄所有对象大小的总和大于 Survivor 空间的一半，则年龄大于或等于该年龄的对象可以直接进入老年代，无需等到 MaxTenuringThreshold 中要求的年龄。

**5. 空间分配担保**

在发生 Minor GC 之前，虚拟机先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果条件成立的话，那么 Minor GC 可以确认是安全的。

如果不成立的话虚拟机会查看 HandlePromotionFailure 设置值是否允许担保失败，如果允许那么就会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次 Minor GC；如果小于，或者 HandlePromotionFailure 设置不允许冒险，那么就要进行一次 Full GC。

#### Full GC 的触发条件: <a href="#fullgc-de-chu-fa-tiao-jian" id="fullgc-de-chu-fa-tiao-jian"></a>

对于 Minor GC，其触发条件非常简单，当 Eden 空间满时，就将触发一次 Minor GC。而 Full GC 则相对复杂，有以下条件:

**1. 调用 System.gc()**

只是建议虚拟机执行 Full GC，但是虚拟机不一定真正去执行。不建议使用这种方式，而是让虚拟机管理内存。

**2. 老年代空间不足**

老年代空间不足的常见场景为前文所讲的大对象直接进入老年代、长期存活的对象进入老年代等。

为了避免以上原因引起的 Full GC，应当尽量不要创建过大的对象以及数组。除此之外，可以通过 -Xmn 虚拟机参数调大新生代的大小，让对象尽量在新生代被回收掉，不进入老年代。还可以通过 -XX:MaxTenuringThreshold 调大对象进入老年代的年龄，让对象在新生代多存活一段时间。

**3. 空间分配担保失败**

使用复制算法的 Minor GC 需要老年代的内存空间作担保，如果担保失败会执行一次 Full GC。具体内容请参考上面的第五小节。

**4. JDK 1.7 及以前的永久代空间不足**

在 JDK 1.7 及以前，HotSpot 虚拟机中的方法区是用永久代实现的，永久代中存放的为一些 Class 的信息、常量、静态变量等数据。

当系统中要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，在未配置为采用 CMS GC 的情况下也会执行 Full GC。如果经过 Full GC 仍然回收不了，那么虚拟机会抛出 java.lang.OutOfMemoryError。

为避免以上原因引起的 Full GC，可采用的方法为增大永久代空间或转为使用 CMS GC。

**5. Concurrent Mode Failure**

执行 CMS GC 的过程中同时有对象要放入老年代，而此时老年代空间不足(可能是 GC 过程中浮动垃圾过多导致暂时性的空间不足)，便会报 Concurrent Mode Failure 错误，并触发 Full GC。
