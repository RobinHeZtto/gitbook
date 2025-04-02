# Android内存管理

### 1. App内存组成以及限制

`Android`给每个`App`分配一个`VM`，让App运行在`dalvik`上，这样即使`App`崩溃也不会影响到系统。系统给`VM`分配了一定的内存大小，`App`可以申请使用的内存大小不能超过此硬性逻辑限制，就算物理内存富余，如果应用超出`VM`最大内存，就会出现内存溢出`crash`。

```
robin@MBP ~ % adb shell getprop  | grep dalvik.vm.heap
[dalvik.vm.heapgrowthlimit]: [256m]
[dalvik.vm.heapmaxfree]: [8m]
[dalvik.vm.heapminfree]: [512k]
[dalvik.vm.heapsize]: [512m]
[dalvik.vm.heapstartsize]: [8m]
[dalvik.vm.heaptargetutilization]: [0.75]
```

**dalvik.vm.heapstartsize** :

堆分配的初始大小。调整这个值会影响到应用的流畅性和整体ram消耗，这个值越小，系统ram消耗越慢，但是由于初始值较小，一些较大的应用需要扩张这个堆，从而引发gc和堆调整的策略，会应用反应更慢。相反，这个值越大系统ram消耗越快，但是程序更流畅。

&#x20;**dalvik.vm.heapgrowthlimit：**  &#x20;

&#x20;极限堆大小，dvm heap是可增长的，但是正常情况下dvm heap的大小是不会超过该值。如果受控的应用dvm heap size超过该值，则将引发oom。

&#x20; **dalvik.vm.heapsize** ：

在manifest中指定android:largeHeap为true时，极限堆大小。一旦dalvik heap size超过这个值，直接引发oom。

**dalvik.vm.heaptargetutilization** ：

可以设定内存利用率的百分比，当实际的利用率偏离这个百分比的时候，虚拟机会在GC的时候调整堆内存大小，让实际占用率向个百分比靠拢。

**dalvik.vm.heapminfree:**

单次Heap内存调整的最小值.

**dalvik.vm.heapmaxfree**:&#x20;

单次Heap内存调整的最大值.

**通过代码获取内存限制：**

```
ActivityManager activityManager = (ActivityManager)context.getSystemService(Context.ACTIVITY_SERVICE)
activityManager.getMemoryClass();//以m为单位
```

**如何修改？**

* 修改 \frameworks\base\core\jni\AndroidRuntime.cpp

```
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
  {
  /*

   * The default starting and maximum size of the heap.  Larger
   * values should be specified in a product property override.
     */
       parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
       parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");//修改这里
     * }
     ...
  }  
```

* 修改 platform/dalvik/+/eclair-release/vm/Init.c

```
gDvm.heapSizeStart = 2 * 1024 * 1024;   // Spec says 16MB; too big for us.
gDvm.heapSizeMax = 16 * 1024 * 1024;    // Spec says 75% physical mem
```

#### 内存指标概念：

| Item | 全称                    | 含义   | 等价                  |
| ---- | --------------------- | ---- | ------------------- |
| USS  | Unique Set Size       | 物理内存 | 进程独占的内存             |
| PSS  | Proportional Set Size | 物理内存 | PSS= USS+ 按比例包含共享库  |
| RSS  | Resident Set Size     | 物理内存 | RSS= USS+ 包含共享库     |
| VSS  | Virtual Set Size      | 虚拟内存 | VSS= RSS+ 未分配实际物理内存 |

总结: VSS >= RSS >= PSS >= USS,但/dev/kgsl-3d0部份必须考虑VSS

### **2**. 低内存杀进程机制

![](<../../../.gitbook/assets/image (160).png>)

* Empty process(空进程)
* Background process(后台进程)
* Service process(服务进程)
* Visible process(可见进程)
* Foreground process(前台进程)

系统需要进行内存回收时最先回收空进程,然后是后台进程，以此类推最后才会回收前台进程（一般情况下前台进程就是与用户交互的进程了,如果连前台进程都需要回收那么此时系统几乎不可用了）。

### 3. Android内存分配与回收

**内存分配：**

Android Dalvik Heap与原生Java一样，也是分代的，这意味着它会根据分配对象的预期寿命和大小跟踪不同的分配存储分区。堆的内存空间分为三个区域，`Young Generation`，`Old Generation`， `Permanent Generation`。

![](<../../../.gitbook/assets/image (104).png>)

分配的对象属于“新生代”。当某个对象保持活动状态达足够长的时间时，可将其提升为较老代，然后是永久代。堆的每一代对相应对象可占用的内存量都有其自身的专用上限。每当一代开始填满时，系统便会执行垃圾回收事件以释放内存。垃圾回收的持续时间取决于它回收的是哪一代对象以及每一代有多少个活动对象。

**1、Young Generation**

由一个Eden区和两个Survivor区组成，程序中生成的大部分新的对象都在Eden区中，当Eden区满时，还存活的对象将被复制到其中一个Survivor区，当次Survivor区满时，此区存活的对象又被复制到另一个Survivor区，当这个Survivor区也满时，会将其中存活的对象复制到年老代。

**2、Old Generation**

一般情况下，年老代中的对象生命周期都比较长。

**3、Permanent Generation**

用于存放静态的类和方法，持久代对垃圾回收没有显著影响。

**总结：**

内存对象的处理过程如下：

* 1、对象创建后在Eden区。
* 2、执行GC后，如果对象仍然存活，则复制到S0区。
* 3、当S0区满时，该区域存活对象将复制到S1区，然后S0清空，接下来S0和S1角色互换。
* 4、当第3步达到一定次数（系统版本不同会有差异）后，存活对象将被复制到Old Generation。
* 5、当这个对象在Old Generation区域停留的时间达到一定程度时，它会被移动到Old Generation，最后累积一定时间再移动到Permanent Generation区域。

系统在Young Generation、Old Generation上采用不同的回收机制。每一个Generation的内存区域都有固定的大小。随着新的对象陆续被分配到此区域，当对象总的大小临近这一级别内存区域的阈值时，会触发GC操作，以便腾出空间来存放其他新的对象。

执行GC占用的时间与Generation和Generation中的对象数量有关：

* Young Generation < Old Generation < Permanent Generation
* Generation中的对象数量与执行时间成正比。

**4、Young Generation GC**

由于其对象存活时间短，因此基于Copying算法（扫描出存活的对象，并复制到一块新的完全未使用的控件中）来回收。新生代采用空闲指针的方式来控制GC触发，指针保持最后一个分配的对象在Young Generation区间的位置，当有新的对象要分配内存时，用于检查空间是否足够，不够就触发GC。

**5、Old Generation GC**

由于其对象存活时间较长，比较稳定，因此采用Mark（标记）算法（扫描出存活的对象，然后再回收未被标记的对象，回收后对空出的空间要么合并，要么标记出来便于下次分配，以减少内存碎片带来的效率损耗）来回收。

**尽管一般垃圾回收速度非常快，但仍会影响应用的性能。**&#x901A;常情况下，您无法从代码中控制何时发生垃圾回收事件。系统有一套专门确定何时执行垃圾回收的标准。当条件满足时，系统会停止执行进程并开始垃圾回收。**如果在动画或音乐播放等密集型处理循环过程中发生垃圾回收，则可能会增加处理时间，进而可能会导致应用中的代码执行超出建议的 16ms 阈值，无法实现高效、流畅的帧渲染。**

> GC发生的时候，线程会被暂停或者抢占。Google在Android8中优化了这个问题， 在ART中对GC过程做了优化，主要是优化了中断和阻塞的时间，但是频繁的GC还是会导致卡顿。
>
> 执行GC所占用的时间和它发生在哪一个Generation也有关系，Young Generation中的每次GC操作时间是最短的，Old Generation其次，Permanent Generation最长。

#### 垃圾回收

ART 有多个不同的 GC 方案，涉及运行不同的垃圾回收器。从 Android 8 (Oreo) 开始，默认方案是并发复制 (CC)。另一个 GC 方案是并发标记清除 (CMS)。

#### 并发压缩式垃圾回收器（Concurrent compacting garbage collector）

ART 在 Android 8.0 中提供了新的并发压缩式垃圾回收器 (GC)。该回收器会在每次执行 GC 时以及应用正在运行时对堆进行压缩，且仅在处理线程根时短暂停顿一次。该回收器具有以下优势：

* GC 始终会对堆进行压缩：堆的大小平均比 Android 7.0 中的小 32%。
* 得益于压缩，系统现可实现线程局部碰撞指针对象分配：分配速度比 Android 7.0 中的快 70%。
* H2 基准的停顿次数比 Android 7.0 GC 的少 85%。
* 停顿次数不再随堆的大小而变化，应用在使用较大的堆时也无需担心造成卡顿。
* GC 实现细节 - 读取屏障：
  * 读取屏障是在读取每个对象字段时所做的少量工作。
  * 它们在编译器中经过了优化，但可能会减慢某些用例的速度。

并发复制 GC 的一些主要特性包括：

* CC 支持使用名为“RegionTLAB”的触碰指针分配器。此分配器可以向每个应用线程分配一个线程本地分配缓冲区 (TLAB)，这样，应用线程只需触碰“栈顶”指针，而无需任何同步操作，即可从其 TLAB 中将对象分配出去。
* CC 通过在不暂停应用线程的情况下并发复制对象来执行堆碎片整理。这是在读取屏障的帮助下实现的，读取屏障会拦截来自堆的引用读取，无需应用开发者进行任何干预。
* GC 只有一次很短的暂停，对于堆大小而言，该次暂停在时间上是一个常量。
* 在 Android 10 及更高版本中，CC 会扩展为分代 GC。它支持轻松回收存留期较短的对象，这类对象通常很快便会无法访问。这有助于提高 GC 吞吐量，并显著延迟执行全堆 GC 的需要。

ART 仍然支持的另一个 GC 方案是 CMS。此 GC 方案还支持压缩，但不是以并发方式。在应用进入后台之前，它会避免执行压缩，应用进入后台后，它会暂停应用线程以执行压缩。如果对象分配因碎片而失败，也必须执行压缩操作。在这种情况下，应用可能会在一段时间内没有响应。

由于 CMS 很少进行压缩，因此空闲对象可能会不连续。CMS 使用一个名为 RosAlloc 的基于空闲列表的分配器。与 RegionTLAB 相比，该分配器的分配成本较高。最后，由于内部碎片，Java 堆的 CMS 内存用量可能会高于 CC 内存用量。

#### 回收策略

CC GC 通过运行新生代 GC 或全堆 GC 来回收垃圾。理想情况下，新生代 GC 的运行频率更高。GC 会一直执行新生代 CC 回收，直到刚结束的回收周期的吞吐量（计算公式是：释放的字节数除以 GC 持续秒数）小于全堆 CC 回收的平均吞吐量。发生这种情况时，将为下一次并发 GC 选择全堆 CC（而不是新生代 CC）。全堆回收完成后，下一次 GC 将切换回新生代 CC。新生代 CC 在完成后不会调整堆占用空间限制，这是此策略发挥作用的一个关键因素。这使得新生代 CC 运行得越来越频繁，直到吞吐量低于全堆 CC，最终导致堆增大。

#### GC类型

* GC\_FOR\_MALLOC: 表示是在堆上分配对象时内存不足触发的GC。
* GC\_CONCURRENT: 当我们应用程序的堆内存达到一定量，或者可以理解为快要满的时候，系统会自动触发GC操作来释放内存。
* GC\_EXPLICIT: 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。
*   GC\_BEFORE\_OOM: 表示是在准备抛OOM异常之前进行的最后努力而触发的GC。

    实际上，**GC\_FOR\_MALLOC、GC\_CONCURRENT和GC\_BEFORE\_OOM三种类型的GC都是在分配对象的过程触发的。而并发和非并发GC的区别主要在于前者在GC过程中，有条件地挂起和唤醒非GC线程，而后者在执行GC的过程中，一直都是挂起非GC线程的。并行GC通过有条件地挂起和唤醒非GC线程，就可以使得应用程序获得更好的响应性。但是同时并行GC需要多执行一次标记根集对象以及递归标记那些在GC过程被访问了的对象的操作，所以也需要花费更多的CPU资源**。后文在ART的并发和非并发GC中我们也会着重说明下这两者的区别。

### 4. 参考资料

* [Android官方-内存管理概览](https://developer.android.com/topic/performance/memory-overview?hl=zh-cn)
* [Android官方-查看堆和内存分配](https://developer.android.com/studio/profile/memory-profiler?hl=zh-cn)
* [Android官方-调试 ART 垃圾回收](https://source.android.com/devices/tech/dalvik/gc-debug)
* [ART运行时垃圾收集机制简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/42072975)
