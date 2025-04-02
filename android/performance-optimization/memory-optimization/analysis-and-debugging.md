# 内存问题分析与定位

### 1. 内存分析指令

常用的内存调优分析命令：

* dumpsys meminfo
* procrank
* cat /proc/meminfo
* free
* showmap
* vmstat

#### 1. dumpsys meminfo

```
robin@MBP ~ % adb shell dumpsys meminfo com.oneplus.note           
Applications Memory Usage (in Kilobytes):
Uptime: 529833295 Realtime: 778714312

** MEMINFO in pid 17334 [com.oneplus.note] **
                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap    11995    11960        0        1    20040    18485     1554
  Dalvik Heap     1106     1088        0        2     3102     1551     1551
 Dalvik Other      306      304        0        1                           
        Stack       64       64        0        0                           
       Ashmem        2        0        0        0                           
      Gfx dev     2172     2172        0        0                           
    Other dev       32        0       32        0                           
     .so mmap     1269       80       20        1                           
    .jar mmap      336        0       12        0                           
    .apk mmap      670        0      132        0                           
    .ttf mmap       33        0        0        0                           
    .dex mmap     4080        8     4072        0                           
    .oat mmap      246        0        0        0                           
    .art mmap     2188     2048        8        2                           
   Other mmap     1599       16      860        0                           
   EGL mtrack    30624    30624        0        0                           
    GL mtrack     5812     5812        0        0                           
      Unknown      807      804        0        1                           
        TOTAL    63349    54980     5136        8    23142    20036     3105
 
 App Summary
                       Pss(KB)
                        ------
           Java Heap:     3144
         Native Heap:    11960
                Code:     4324
               Stack:       64
            Graphics:    38608
       Private Other:     2016
              System:     3233
 
               TOTAL:    63349       TOTAL SWAP PSS:        8
 
 Objects
               Views:       63         ViewRootImpl:        1
         AppContexts:        6           Activities:        1
              Assets:       21        AssetManagers:        0
       Local Binders:       23        Proxy Binders:       32
       Parcel memory:        4         Parcel count:       16
    Death Recipients:        1      OpenSSL Sockets:        0
            WebViews:        0
 
 SQL
         MEMORY_USED:      165
  PAGECACHE_OVERFLOW:       19          MALLOC_SIZE:      117
 
 DATABASES
      pgsz     dbsz   Lookaside(b)          cache  Dbname
         4       28             60         1/16/2  /data/user/0/com.oneplus.note/databases/note.db
```

相关参数的说明：

**Pss Total**：**进程实际使用的内存，该统计方法包括比例分配共享库占用的内存**，即如果有三个进程共享了一个共享库，则平摊分配该共享库占用的内存。Pss Total统计方法的一个需要注意的地方是如果使用共享库的一个进程被杀死，则共享库的内存占用按比例分配到其他共享该库的进程中，而不是将内存资源返回给系统，这种情况下PssTotal不能够准确代表内存返回给系统的情况。

**Private Dirty**：进程私有的脏页内存大小，该统计方法只包括进程私有的被修改的内存。

**Private Clear：**&#x8FDB;程私有的干净页内存大小，该统计方法只包括进程私有的没有被修改的内存。

**Swapss Dirty：**&#x88AB;交换的脏页内存大小，该内存与其他进程共享。

> **其中private Dirty + private Clean = Uss**，**该值是一个进程的使用的私有内存大小，即这些内存唯一被该进程所有。该统计方法真正描述了运行一个进程需要的内存和杀死一个进程释放的内存情况，是怀疑内存泄露最好的统计方法。**
>
> 共享比例：sharing\_proportion = (Pss Total - private\_clean - private\_dirty) / (shared\_clean + shared\_dirty)
>
> 能够被共享的内存：swappable\_pss = (sharing\_proportion \* shared\_clean) + private\_clean

**Native Heap：**&#x672C;地堆使用的内存，包括C/C++在堆上分配的内存

**Dalvik Heap：**&#x64;alvik虚拟机使用的内存

**Dalvik other：**&#x9664;Dalvik和Native之外分配的内存，包括C/C++分配的非堆内存

**Cursor：**&#x6570;据库游标文件占用的内存

**Ashmem：**&#x533F;名共享内存

**Stack：**&#x44;alvik栈占用的内存

**Other dev**：其他的dev占用的内存

**.so mmap：**&#x73;o库占用的内存

**.jar mmap**：.jar文件占用的内存

.**apk mmap：**.apk文件占用的内存

**.ttf mmap：**.ttf文件占用的内存

**.dex mmap：**.dex文件占用的内存

**image mmap：**&#x56FE;像文件占用的内存

**code mmap：**&#x4EE3;码文件占用的内存

**Other mmap：**&#x5176;他文件占用的内存

**Graphics：**&#x47;PU使用图像时使用的内存

**GL：**&#x47;PU使用GL绘制时使用的内存

**Memtrack：**&#x47;PU使用多媒体、照相机时使用的内存

**Unknown：**&#x4E0D;知道的内存消耗

**Heap Size：**&#x5806;的总内存大小

**Heap Alloc：**&#x5806;分配的内存大小

**Heap Free：**&#x5806;待分配的内存大小

**Native Heap | Heap Size :** 从mallinfo usmblks获的，当前进程Native堆的最大总共分配内存

**Native Heap | Heap Alloc :** 从mallinfo uorblks获的，当前进程navtive堆的总共分配内存

**Native Heap | Heap Free :** 从mallinfo fordblks获的，当前进程Native堆的剩余内存

> Native Heap Size ≈ Native Heap Alloc + Native Heap Free，mallinfo是一个C库，mallinfo()函数提供了各种各样通过malloc()函数分配的内存的统计信息。

**Dalvik Heap | Heap Size :** 从Runtime totalMemory()获得，Dalvik Heap总共的内存大小

**Dalvik Heap | Heap Alloc :** 从Runtime totalMemory() - freeMemory()获得，Dalvik Heap分配的内存大小

**Dalvik Heap | Heap Free :** 从Runtime freeMemory()获得，Dalvik Heap剩余的内存大小

> Dalvik Heap Size = Dalvik Heap Alloc + Dalvik Heap Free

**Obejcts:** 统计当前进程中的对象个数

**Views:** 当前进程中实例化的视图View对象数量

**ViewRootImpl:** 当前进程中实例化的视图根ViewRootImpl对象数量

**AppContexts:** 当前进程中实例化的应用上下文ContextImpl对象数量

**Activities:** 当前进程中实例化的Activity对象数量

**Assets:** 当前进程的全局资产数量

**AssetManagers:** 当前进程的全局资产管理数量

**Local Binders:** 当前进程有效的本地binder对象数量

**Proxy Binders:** 当前进程中引用的远程binder对象数量

**Death Recipients:** 当前进程到binder的无效链接数量

**OpenSSL Sockets:** 安全套接字对象数量

**SQL包括：**

**MEMORY\_USED:** 当前进程中数据库使用的内存数量，kb

**PAGECACHE\_OVERFLOW:** 页面缓存的配置不能够满足的数量，kb

**MALLOC\_SIZE:** 向sqlite3请求的最大内存分配数量，kb

**DATABASES包括：**

**pgsz:** 数据库的页面大小

**dbsz:** 数据库大小

**Lookaside(b):** 后备使用的内存大小

**cache:** 数据缓存状态

**Dbname:** 数据库表名

#### 2. procrank

功能： 获取所有进程的内存使用的排行榜，排行是以`Pss`的大小而排序。`procrank`命令比`dumpsys meminfo`命令，能输出更详细的VSS/RSS/PSS/USS内存指标。

最后一行输出下面6个指标：

| total    | free    | buffers | cached | shmem | slab   |
| -------- | ------- | ------- | ------ | ----- | ------ |
| 2857032K | 998088K | 78060K  | 78060K | 312K  | 92392K |

执行结果：

```
root@Phone:/# procrank
  PID       Vss      Rss      Pss      Uss  cmdline
 4395  2270020K  202312K  136099K  121964K  com.android.systemui
 1192  2280404K  147048K   89883K   84144K  system_server
29256  2145676K   97880K   44328K   40676K  com.android.settings
  501  1458332K   61876K   23609K    9736K  zygote
 4239  2105784K   68056K   21665K   19592K  com.android.phone
  479   164392K   24068K   17970K   15364K  /system/bin/mediaserver
  391   200892K   27272K   15930K   11664K  /system/bin/surfaceflinger
...
RAM: 2857032K total, 998088K free, 78060K buffers, c cached, 312K shmem, 92392K slab
```

#### 3. cat /proc/meminfo

功能：能否查看更加详细的内存信息

```
指令： cat /proc/meminfo
```

输出结果如下(结果内存值不带小数点，此处添加小数点的目的是为了便于比对大小)：

```
root@phone:/ # cat /proc/meminfo
MemTotal:        2857.032 kB  //RAM可用的总大小 (即物理总内存减去系统预留和内核二进制代码大小)
MemFree:         1020.708 kB  //RAM未使用的大小
Buffers:           75.104 kB  //用于文件缓冲
Cached:           448.244 kB  //用于高速缓存
SwapCached:             0 kB  //用于swap缓存
​
Active:           832.900 kB  //活跃使用状态，记录最近使用过的内存，通常不回收用于其它目的
Inactive:         391.128 kB  //非活跃使用状态，记录最近并没有使用过的内存，能够被回收用于其他目的
Active(anon):     700.744 kB  //Active = Active(anon) + Active(file)
Inactive(anon):       228 kB  //Inactive = Inactive(anon) + Inactive(file)
Active(file):     132.156 kB
Inactive(file):   390.900 kB
​
Unevictable:            0 kB
Mlocked:                0 kB
​
SwapTotal:        524.284 kB  //swap总大小
SwapFree:         524.284 kB  //swap可用大小
Dirty:                  0 kB  //等待往磁盘回写的大小
Writeback:              0 kB  //正在往磁盘回写的大小
​
AnonPages:        700.700 kB  //匿名页，用户空间的页表，没有对应的文件
Mapped:           187.096 kB  //文件通过mmap分配的内存，用于map设备、文件或者库
Shmem:               .312 kB
​
Slab:              91.276 kB  //kernel数据结构的缓存大小，Slab=SReclaimable+SUnreclaim
SReclaimable:      32.484 kB  //可回收的slab的大小
SUnreclaim:        58.792 kB  //不可回收slab的大小
​
KernelStack:       25.024 kB
PageTables:        23.752 kB  //以最低的页表级
NFS_Unstable:           0 kB  //不稳定页表的大小
Bounce:                 0 kB
WritebackTmp:           0 kB
CommitLimit:     1952.800 kB
Committed_AS:   82204.348 kB   //评估完成的工作量，代表最糟糕case下的值，该值也包含swap内存
​
VmallocTotal:  251658.176 kB  //总分配的虚拟地址空间
VmallocUsed:      166.648 kB  //已使用的虚拟地址空间
VmallocChunk:  251398.700 kB  //虚拟地址空间可用的最大连续内存块
```

对于cache和buffer也是系统可以使用的内存。所以系统总的可用内存为 MemFree+Buffers+Cached

#### 4. free

主功能：查看可用内存，缺省单位KB。该命令比较简单、轻量，专注于查看剩余内存情况。数据来源于/proc/meminfo。

输出结果：

```
root@phone:/proc/sys/vm # free
             total         used         free       shared      buffers
Mem:       2857032      1836040      1020992            0        75104
-/+ buffers:            1760936      1096096
Swap:       524284            0       524284
```

* 对于`Mem`行，存在的公式关系： total = used + free;
* 对于`-/+ buffers`行： 1760936 = 1836040 - 75104(buffers); 1096096 = 1020992 + 75104(buffers);

#### 5. showmap

主功能：用于查看虚拟地址区域的内存情况

```
用法：  showmap -a [pid]
```

该命令的输出每一行代表一个虚拟地址区域(vm area)

* start addr和end addr:分别代表进程空间的起止虚拟地址；
* virtual size/ RSS /PSS这些前面介绍过；
* shared clean：代表多个进程的虚拟地址可指向这块物理空间，即有多少个进程共享这个库；
* shared: 共享数据
* private: 该进程私有数据
* clean: 干净数据，是指该内存数据与disk数据一致，当内存紧张时，可直接释放内存，不需要回写到disk
* dirty: 脏数据，与disk数据不一致，需要先回写到disk，才能被释放。

#### 6. vmstat

主功能：不仅可以查看内存情况，还可以查看进程运行队列、系统切换、CPU时间占比等情况，另外该指令还是周期性地动态输出。

用法：

```
Usage: vmstat [ -n iterations ] [ -d delay ] [ -r header_repeat ]
    -n iterations     数据循环输出的次数
    -d delay          两次数据间的延迟时长(单位：S)
    -r header_repeat  循环多少次，再输出一次头信息行
```

输入结果：

```
root@phone:/ # vmstat
procs  memory                       system          cpu
 r  b   free  mapped   anon   slab    in   cs  flt  us ni sy id wa ir
 2  0  663436 232836 915192 113960   196  274    0   8  0  2 99  0  0
 0  0  663444 232836 915108 113960   180  260    0   7  0  3 99  0  0
 0  0  663476 232836 915216 113960   154  224    0   2  0  5 99  0  0
 1  0  663132 232836 915304 113960   179  259    0  11  0  3 99  0  0
 2  0  663124 232836 915096 113960   110  175    0   4  0  3 99  0  0
```

参数列总共15个参数，分为4大类：

* procs(进程)
  * r: Running队列中进程数量
  * b: IO wait的进程数量
* memory(内存)
  * free: 可用内存大小
  * mapped：mmap映射的内存大小
  * anon: 匿名内存大小
  * slab: slab的内存大小
* system(系统)
  * in: 每秒的中断次数(包括时钟中断)
  * cs: 每秒上下文切换的次数
* cpu(处理器)
  * us: user time
  * ni: nice time
  * sy: system time
  * id: idle time
  * wa: iowait time
  * ir: interrupt time

#### 总结

1. `dumpsys meminfo`适用场景： 查看进程的oom adj，或者dalvik/native等区域内存情况，或者某个进程或apk的内存情况，功能非常强大；
2. `procrank`适用场景： 查看进程的VSS/RSS/PSS/USS各个内存指标；
3. `cat /proc/meminfo`适用场景： 查看系统的详尽内存信息，包含内核情况；
4. `free`适用场景： 只查看系统的可用内存；
5. `showmap`适用场景： 查看进程的虚拟地址空间的内存分配情况；
6. `vmstat`适用场景： 周期性地打印出进程运行队列、系统切换、CPU时间占比等情况；



### 2. Memory Analyzer (MAT)

#### 重要概念：

**Heap Dump：**

&#x20;       Heap Dump是Java进程在某个时刻的**内存快照**，不同JVM的实现的Heap Dump的文件格式可能不同，进而存储的数据也可能不同，但是一般来说。Heap Dump中主要包含当生成快照时堆中的java对象和类的信息，主要分为如下几类：

* 对象信息：类名、属性、基础类型和引用类型
* 类信息：类加载器、类名称、超类、静态属性
* gc roots：JVM中的一个定义，进行垃圾收集时，要遍历可达对象的起点节点的集合
* 线程栈和局部变量：快照生成时候的线程调用栈，和每个栈上的局部变量

&#x20;       Heap Dump中没有包含对象的分配信息，因此它不能用来分析这种问题：一个对象什么时候被创建、一个对象时被谁创建的。

**Shallow Heap & Retained Heap：**

&#x20;       Shallow heap是一个**对象本身占用的堆内存大小**。一个对象中，每个引用占用8或64位，Integer占用4字节，Long占用8字节等等。

&#x20;       在了解Retained heap之前，首先需要了解一下Retained set的概念。对于某个对象X来说，它的Retained set指的是，如果对象X被垃圾收集器回收了，那么Retained set这个集合中的对象都会被回收，如果X没有被垃圾收集器回收，那么这个Retained set集合中的对象都不会被回收。

&#x20;       Retained heap指的时候它的Retained set中的所有对象与Shallow heap的总和，换句话说，Retained heap指的是对象X的保留内存大小，**即由于它的存活导致多大的内存也没有被回收**。

![](<../../../.gitbook/assets/image (132).png>)

**incoming references & outgoing references：**

* 对象 A 和对象 B 持有对象 C 的引用
* 对象 C 持有对象 D 和对象 E 的引用

![](<../../../.gitbook/assets/image (127).png>)

**拥有对象 C 的引用的所有对象都称为 Incoming references。**&#x5BF9;象 C 的“Incoming references”是对象 A、对象 B 和 **C 的类对象** 。

**对象 C 引用的所有对象都称为 Outgoing References。**&#x5BF9;象 C 的“outgoing references”是对象 D、对象 E 和 C 的类对象。

#### MAT界面：

&#x31;**.首页功能：**

![](<../../../.gitbook/assets/image (479).png>)

1. inspector：透视图，用于展示一个对象的详细信息，例如内存地址、加载器名称、包名、对象名称、对象所属的类的父类、对象所属的类的加载器对象、该对象的堆内存大小和保留大小，gc root信息。
2. inspector窗口的下半部分：展示类的静态属性和值、对象的实例属性和值、对象所属的类的继承结构。
3. Heap Dump History：用于列举最近分析过的文件
4. 常用功能栏：从左到右依次是：概览、类直方图、支配树、OQL查询、线程视图、报告相关、详细功能。其中概览就是在刚解析完后展示的这个页面，详细功能按钮则是提供了一些更细致的分析能力。
5. 概览中的饼图：该饼图用于展示retained size最大的对象。
6. 常用的分析动作：类直方图、支配树、按照类和包路径获取消耗资源最多的对象、重名类。
7. 报告相关：Leak Suspects用于查找内存泄漏问题，以及系统概览
8. Components Report：这个功能是一组功能的集合，用于分析某一类性的类的实例的问题，例如分析`java.util.*`开头的类的实例对象的一些使用情况，例如：重复字符串、空集合、集合的使用率、软引用的统计、finalizer的统计、Map集合的碰撞率等等。

**2. 配置页面：**

`Window` -> `Preferences`

![](<../../../.gitbook/assets/image (472).png>)

配置选项：

* **Keep unreachable objects：**&#x5982;果勾选这个，则在分析的时候会包含dump文件中的不可达对象；
* **Bytes Display：**&#x8BBE;置分析结果中内存大小的展示单位

#### **类直方图:**

&#x20;       直方图是从类的角度看哪些类及该类的实例对象占用着内存情况，默认是按照某个类的shallow heap大小从大到小排序。

![](<../../../.gitbook/assets/image (165).png>)

&#x20;       Retained Heap这一列的值可能是显示`>=Shallow Heap数值`，因为对于某个类的所有实例计算总的retained heap非常慢，因此右键选择快速估算或者精确计算。

![](<../../../.gitbook/assets/image (264).png>)

&#x20;       另外一个重要的功能，可以选择从不同的维度进行分类分析，例如superclass、class loader、package。

![](<../../../.gitbook/assets/image (163).png>)

#### Dominator Tree：

&#x20;       MAT根据堆上的**对象引用关系**构建了支配树（Dominator Tree），通过支配树可以很方便得识别出哪些对象占用了大量的内存，并可以看到它们之间的依赖关系。跟类直方图类似，支配树页面也可以选择从类加载器、package等维度来查看。

![](<../../../.gitbook/assets/image (401).png>)



#### **线程视图：**

&#x20;       在线程视图这个表中，可以看到以下几个信息：线程对象的名字、线程名、线程对象占用的堆内存大小、线程对象的保留堆内存大小、线程的上下文加载器、是否为守护线程。

![](<../../../.gitbook/assets/image (470).png>)

&#x20;       选中某个线程对象展开，可以看到线程的调用栈和每个栈的局部变量，通过查看线程的调用栈和局部变量的内存大小，可以找到在哪个调用栈里分配了大量的内存。

### 3. Android Studio Memory-profiler

{% embed url="https://developer.android.com/studio/profile/memory-profiler#performance" %}

### 4. LeakCanary

{% embed url="https://github.com/square/leakcanary" %}



### 5. 内存泄漏常见场景以及解决方案

#### 1、资源性对象未关闭

对于资源性对象不再使用时，应该立即调用它的close()函数，将其关闭，然后再置为null。例如Bitmap等资源未关闭会造成内存泄漏，此时我们应该在Activity销毁时及时关闭。

#### 2、注册对象未注销

例如BraodcastReceiver、EventBus未注销造成的内存泄漏，我们应该在Activity销毁时及时注销。

#### 3、类的静态变量持有大数据对象

尽量避免使用静态变量存储数据，特别是大数据对象，建议使用数据库存储。

#### 4、单例造成的内存泄漏

优先使用Application的Context，如需使用Activity的Context，可以在传入Context时使用弱引用进行封装，然后，在使用到的地方从弱引用中获取Context，如果获取不到，则直接return即可。

#### 5、非静态内部类的静态实例

该实例的生命周期和应用一样长，这就导致该静态实例一直持有该Activity的引用，Activity的内存资源不能正常回收。此时，我们可以将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，尽量使用Application Context，如果需要使用Activity Context，就记得用完后置空让GC可以回收，否则还是会内存泄漏。

#### 6、Handler临时性内存泄漏

Message发出之后存储在MessageQueue中，在Message中存在一个target，它是Handler的一个引用，Message在Queue中存在的时间过长，就会导致Handler无法被回收。如果Handler是非静态的，则会导致Activity或者Service不会被回收。并且消息队列是在一个Looper线程中不断地轮询处理消息，当这个Activity退出时，消息队列中还有未处理的消息或者正在处理的消息，并且消息队列中的Message持有Handler实例的引用，Handler又持有Activity的引用，所以导致该Activity的内存资源无法及时回收，引发内存泄漏。解决方案如下所示：

* 1、使用一个静态Handler内部类，然后对Handler持有的对象（一般是Activity）使用弱引用，这样在回收时，也可以回收Handler持有的对象。
* 2、在Activity的Destroy或者Stop时，应该移除消息队列中的消息，避免Looper线程的消息队列中有待处理的消息需要处理。

需要注意的是，AsyncTask内部也是Handler机制，同样存在内存泄漏风险，但其一般是临时性的。对于类似AsyncTask或是线程造成的内存泄漏，我们也可以将AsyncTask和Runnable类独立出来或者使用静态内部类。

#### 7、容器中的对象没清理造成的内存泄漏

在退出程序之前，将集合里的东西clear，然后置为null，再退出程序

#### 8、WebView

WebView都存在内存泄漏的问题，在应用中只要使用一次WebView，内存就不会被释放掉。我们可以为WebView开启一个独立的进程，使用AIDL与应用的主进程进行通信，WebView所在的进程可以根据业务的需要选择合适的时机进行销毁，达到正常释放内存的目的。

#### 9、使用ListView时造成的内存泄漏

在构造Adapter时，使用缓存的convertView。

### 6. OOM

![](<../../../.gitbook/assets/image (4).png>)

西瓜视频tailor：[https://github.com/bytedance/tailor](https://github.com/bytedance/tailor) 内存快照裁剪，回捞，<10MB

快手KOOM：[https://github.com/KwaiAppTeam/KOOM](https://github.com/KwaiAppTeam/KOOM) 本地分析，报告上传

美团Probe:

策略：&#x20;

线下: 开发、回归、Monkey、压测等环节，自动集成 LeakCanary 检测内存泄漏；&#x20;

线上: 回捞快照精准分析。

### 7. 参考资料

* [美团Android OOM案例分析](https://tech.meituan.com/2017/04/14/oom-analysis.html)
* [Probe：Android线上OOM问题定位组件](https://tech.meituan.com/2019/11/14/crash-oom-probe-practice.html)
* [西瓜视频稳定性治理体系建设一：Tailor 原理及实践](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==\&mid=2247487203\&idx=1\&sn=182584b69910c843ae95f60e74127249\&chksm=e9d0c501dea74c178e16f95a2ffc5007c5dbca89a02d56895ed9b05883cf0562da689ac6146b\&mpshare=1\&scene=1\&srcid=1214J5eqf7BLos4XUPVNwHaZ\&sharer_sharetime=1607920220074\&sharer_shareid=942119afdfbc37ad9eb04201dfe5b060\&key=c625067ba47b16f23b52ab1c6fea072733f830cc075ec2ef55903da51f5e84a2e5ed54364b574707142442642a1044cbe553d88fb5d5f78bf5a6dfb73125a89be47fd6bdfec2ef15aa6e7079aaec70d978d516dada395a952eaed741a8530550fef692c895c9f5d1f5b0503da684ff9436a78de5696b3b5bd831b650dae72760\&ascene=1\&uin=NDY1Mzg4MTg4\&devicetype=Windows+10+x64\&version=63000039\&lang=zh_CN\&exportkey=A1IUb4RZzDsgZbh%2BD%2BrEKJw%3D\&pass_ticket=nIPY2niCmsClO%2BEdQ8nnrIVvUrmsypwYQX1WdqDm%2FHKJlWTvMGbSGxy5Xqh289xx\&wx_header=0)
* [快手开源自研OOM解决方案KOOM](https://www.infoq.cn/article/czH2vGc5wOjTXXo19WPc)
* [Android 内存优化总结&实践](https://mp.weixin.qq.com/s/2MsEAR9pQfMr1Sfs7cPdWQ)
