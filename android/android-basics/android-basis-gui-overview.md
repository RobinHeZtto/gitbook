# Android基础之GUI系统概览

> 图形用户界面（英语：**G**raphical **U**ser **I**nterface，缩写：**GUI**）是指采用图形方式显示的计算机操作用户界面。

Android的GUI系统是Android最重要也最复杂的系统之一。我们在日常开发中，除了经常使用的Activity、View等对象或组件以外，可能很少会关注到GUI系统的运行机制及底层的实现逻辑。对于很多的原理性问题，可能只理解部分或者完全不了解，知其然不知其所以然，造成很多概念模邻二可的情况。所以，了解整个GUI系统的运作，可以让我们在开发过程中站在更高的角度思考问题。即所谓 _“会当凌绝顶，一览众山小。”_

在应用开发过程中，我们可能会有很多疑问:

1. 我们定义的一个个的控件是怎么画出来的？然后又是怎么显示到手机屏幕上的？
2. Window是什么？Activity跟Window的关系是什么？在Activity的背后系统都帮我们做了哪些事情？
3. 窗口是怎么获取焦点的？又是怎么响应用户事件的？
4. Surface是什么？是谁帮我们创建的？
5. ......

下面先了解Android GUI的设计原理，后面的一系列总结中将对Android GUI的设计有更加深入的了解，一步步解开我们心中的疑惑。

## 1. 手机GUI的发展

![](<../../.gitbook/assets/image (35).png>)

![](<../../.gitbook/assets/image (327).png>)

在21世纪初期，我们所使用的手机基本上都是下面的键盘手机，在那个时代，几乎没有**app**、**应用程序**的概念，不同的手机可能使用的都是不同的操作系统，更不用提什么“**系统生态**”。

![](<../../.gitbook/assets/image (39).png>)

直到他的出现，改变了整个世界。2017年iPhone的发布，是一个里程碑，是一条历史分界线，手机GUI发展的这段不长的时间里，可以粗暴的区别为：**iPhone前时代**与**iPhone后时代**。

![](<../../.gitbook/assets/image (392).png>)

随后，带着互联网精神的安卓系统出现，**Google**免费开放了它的源代码，使得任意公司均可对安卓系统的UI界面以及功能方面的东西进行更改，Android在各大厂商的定制修改下百花齐放。

![](<../../.gitbook/assets/image (406).png>)

在后续移动互联网爆发式的发展中，移动设备的GUI也变得更加美观，交互也更加顺畅。

![](<../../.gitbook/assets/image (358).png>)

## 2. C/S 架构

无论是 Android、iOS、还是 MacOS、Windows、Linux，GUI系统为了实现输入和输出的完美协调，必然需要基于 C/S 架构。

为什么呢？

整个GUI视图系统主要包括**输入**与**输出**二个部分，输出即**排版、渲染、显示图像**，输入即**事件收集、分发响应**。其中排版结果的渲染显示 和 屏幕事件的接收，是整个系统通用的、且使用频率极高，所以它应当在处于支持高效率运行的、规避数据被篡改或破坏的环境下运作。而排版与事件响应的操作每个应用千差万别，形态各异，它应当可以方便的定制，以向用户提供适合各种场景的UI交互。

由此我们的GUI系统 便不得不分为两部分来运行，一部分是常驻在系统进程中的**视图服务**，负责渲染显示与事件收集。另一部分就是运行在用户自己开启进程中的 **视图接口（客户端）**，负责UI排版和响应事件，它们之间都是通过这个“接口”与视图服务交互的。

![](<../../.gitbook/assets/image (12).png>)

* 让使用频率极高的视图服务 常驻系统进程，以减少频繁创建进程导致的开销，以及由于是在系统进程中运行，因而效率更高、更安全。
* 与此同时，让客户端 在用户进程中生死存亡，从而达到即用即走、节省系统资源的目的。

而 **正是因为这个需要，造成了 视图接口 和 视图服务 分别处于不同的进程，从而有 IPC（Inter-Process Communication，进程间通信）的需要**。而该场景下效率最高的 IPC 通信方式是 RPC（Remote Procedure Call，远程过程调用）：一种 C/S 架构的实现，因而，视图系统需要基于 C/S 架构。

![](<../../.gitbook/assets/image (239).png>)

**视图接口的抽象：**

为了 传递排版结果 和 接收触控事件，在 视图接口端（客户端），必然需要存在 负责排版结果 和 接收触控事件 的 “窗口配置” 实体。

具体而言，一方面 它需要存在一套规则，来 **支持对视图树的递归**，从而完成用于输出的排版工作；另一方面，它需要存在一套接口（interface），来 **约定** 规格和入口，从而完成输入事件从服务到接口端 的传递。即该实体承担了 **排版属性的存储** 和 **窗口事件的代理** 两大职责。

![](<../../.gitbook/assets/image (10).png>)

**视图服务的抽象：**

视图服务中，我们需要 **根据客户端传递的窗口配置，来生成对应的窗口实体**，从而 根据 **该实体所持有的排版结果的描述，来最终渲染在屏幕上**。

![](<../../.gitbook/assets/image (99).png>)

## 3. 基本概念

在了解Android GUI系统之前，先了解一下该领域的一些专业术语或基础概念。

**渲染(Render)：**

渲染（render）是指软件模型生成图像的过程。其中软件模型是指用语言或者数据结构定义的三维物体或虚拟场景的描述，它包括几何、视点、纹理、照明和阴影等信息，图像是数字图像或者位图图像。

**栅格化(Rasterization)：**

栅格化(Rasterization)是指将向量图形格式表示的图像转换成位图（像素）以用于显示设备输出的过程，简单来说就是将我们要显示的视图，转换成用像素来表示的格式。栅格化是一个非常耗时的工作。

**CPU：**

CPU（Central Processing Unit，中央处理器），是计算机系统的运算和控制核⼼，是信息 处理、程序运⾏的最终执⾏单元。

**GPU：**

GPU（Graphics Processin Unit，图形处理器），是⼀种专⻔⽤于图像运算的处理器，在计算机系统中通常被称为 "显卡"的核⼼部件就是 GPU。

> CPU 更擅长复杂逻辑控制，而 GPU 得益于大量 ALU 和并行结构设计，更擅长数学运算

**软件绘制、硬件绘制：**

软件绘制是指所有的图形计算工作全部由CPU独立完成，而硬件绘制是通过底层软件代码，将 CPU 不擅长的图形计算转换成 GPU 专用指令，由 GPU 完成绘制任务。

![](<../../.gitbook/assets/image (15).png>)

**OpenGL：**

OpenGL是Khronos Group开发维护的一个规范，它主要为我们定义了用来操作图形和图片的一系列函数的API，需要注意的是OpenGL本身并非API。GPU的硬件开发商则需要提供满足OpenGL规范的实现，这些实现通常被称为“驱动”，它们负责将OpenGL定义的API命令翻译为GPU指令。

> GPU 作为一个硬件，用户空间是无法直接使用的，它通过由 GPU 厂商按照 OpenGL 规范实现的驱动程序间接进行使用。

**OpenGL ES：**

OpenGL ES (Embedded Systems）是针对嵌入式设备的 OpenGL 规范的一种变体。Android 支持多个版本的 OpenGL ES API。

**Vulkan：**

Vulkan是一个低开销、跨平台的二维、三维图形与计算的应用程序接口（API），由作OpenGL官方组织Khronos推出的新一代API规范，它的设计目标是取代 OpenGL，相比于OPenGL，同样采用跨平台设计，但最重要的贡献是大幅降低绘制命令开销（draw call overhead），改善多线程性能，渲染性能更快。

**Skia：**

skia是个2D向量图形处理函数库，包含字型、坐标转换，以及点阵图都有高效能且简洁的表现。Skia Graphics Library（SGL）由C++编写，最初由Skia公司开发，被Google收购后以New BSD License许可下开源。Skia能在低端设备如手机上呈现高质量的2D图形。截至2017年，它已被应用于Android、Google Chrome、Chrome OS、Chromium OS、Mozilla Firefox、Firefox OS以及Sublime Text。

**刷新率：**

刷新率是指显示器每秒显示新图像的次数。

**帧率：**

帧率是指每秒显示的帧数，

## 4. Android GUI组成

Android基于Linux开发，GUI系统包括输入及显示而大部分。

![](<../../.gitbook/assets/image (370).png>)

### 4.1 显示系统

先通过下图感性了解一下Android 显示系统，以图中间那条线为界，将显示系统划分为上下两层。 上层为应用级别的显示，解决如何绘制渲染图层的问题。 下层为系统级别的显示，解决如何将绘制好的图层送显的问题。

![](<../../.gitbook/assets/image (50).png>)

#### 应用侧绘制：

一个 Android 应用程序窗口里面包含了很多 UI 元素，它们是以树形结构来组织的，父视图包含子视图，一个窗口的根视图是 DecorView 对象。

在绘制一个 Android 应用程序窗口的 UI 之前，我们首先要确定它里面的各个子 UI 元素在父 UI 元素里面的大小以及位置。确定各个子 UI 元素在父 UI 元素里面的大小以及位置的过程又称为测量过程和布局过程。因此，Android 应用程序窗口的 UI 渲染过程可以分为测量、布局和绘制三个阶段，最后生成 **Display List**。

如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201209141512679.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1Nzg3MzQ=,size_16,color_FFFFFF,t_70)

1. 测量：递归（深度优先）确定所有视图的大小（高、宽）
2. 布局：递归（深度优先）确定所有视图的位置（左上角坐标）
3. 绘制：在画布 canvas 上绘制应用程序窗口所有的视图
4. 生成 Display List 数据。

#### 系统侧渲染 :

**1. Display List 数据交由 GPU 进行渲染处理**

Android 需要把 XML 布局文件转换成 GPU 能够识别并绘制的对象。这个操作是在 DisplayList 的帮助下完成的。Display List 持有所有将要交给 GPU 绘制到屏幕上的数据信息。

Display List 是一个缓存绘制命令的 Buffer，Display List 的本质是一个缓冲区，它里面记录了即将要执行的绘制命令序列。**渲染 Display List，发生在应用程序进程的 Render Thread 中**。增加 Render Thread 线程，也是为了避免 UI 线程任务过重，用于提高渲染性能。**这些绘制命令最终会转化为 Open GL 命令由 GPU 执行**。这意味着我们在调用 Canvas API 绘制 UI 时，实际上只是将 Canvas API 调用及其参数记录在 Display List 中，然后等到下一个 VSYNC 信号到来时，记录在 Display List 里面的绘制命令才会转化为 Open GL 命令由 GPU 执行。

**2. GPU 渲染处理**

Android 使用 OpenGL ES (GLES) API 渲染图形。GPU 将视图栅格化后，生成 Surface。GPU 作为图像流生产方将显示内容，最终发送给图像流的消耗方 SurfaceFlinger。

大多数客户端使用 OpenGL ES 或 Vulkan 渲染到 Surface 上（硬件加速，使用了 GPU 渲染）。但是，有些客户端使用画布渲染到 Surface 上（未使用硬件加速）。

**3. 生成 Surface 并存储到 BufferQueue**

Surface 对象使应用能够渲染要在屏幕上显示的图像。通过 SurfaceHolder 接口，应用可以编辑和控制 Surface。Surface 是一个接口，供生产方与使用方交换缓冲区。

SurfaceHolder 是系统用于与应用共享 Surface 所有权的接口。与 Surface 配合使用的一些客户端需要 SurfaceHolder，因为用于获取和设置 Surface 参数的 API 是通过 SurfaceHolder 实现的。一个 SurfaceView 包含一个 SurfaceHolder。与 View 交互的大多数组件都涉及到 SurfaceHolder。如果你开发的是相机相关的应用，会涉及到相关 API 的使用。

通常 SurfaceFlinger 使用的缓冲区队列是由 Surface 生成的，当渲染到 Surface 上时，结果最终将出现在传送给消费者的缓冲区中。Canvas API 提供一种软件实现方法（支持硬件加速），用于直接在 Surface 上绘图（OpenGL ES 的低级别替代方案）。与视图有关的任何内容均涉及到 SurfaceHolder，其 API 可用于获取和设置 Surface 参数（如大小和格式）。

BufferQueue 是 SurfaceFlinger 使用的缓冲区队列，而 Surface 是 BufferQueue 的生产方。BufferQueue 类将可生成图形数据缓冲区的组件（生产方）连接到接受数据以便进行显示或进一步处理的组件（使用方，例如 SurfaceFlinger）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。

用于显示 Surface 的 BufferQueue 通常配置为三重缓冲。缓冲区是按需分配的，因此，如果生产方足够缓慢地生成缓冲区（例如在 60 fps 的显示屏上以 30 fps 的速度进行缓冲），队列中可能只有两个分配的缓冲区。按需分配缓冲区有助于最大限度地减少内存消耗。

数据处理过程：

![](<../../.gitbook/assets/image (58).png>)

上图描述了图形管道的流程。左侧为图形生产者，右侧为图形消费者，中间通过 BufferQueues 连接。图中，主屏幕、状态栏和系统界面通过 GPU 渲染生成图形缓冲区，做为生产者传递给 BufferQueues。SurfaceFlinger 做为消费者，接收到 BufferQueues 的通知后，取出可用的图形缓冲区，送给显示端。例图中将状态栏和系统界面的图形缓冲送给 GPU 合成，生成新的图形缓冲后再通过 BufferQueues 发送到硬件混合渲染器。

注意：SurfaceFlinger 创建并拥有 BufferQueue 数据结构，并且可存在于与其生产方不同的进程中。

**４. SurfaceFlinger 将显示数据发送给显示屏**

SurfaceFlinger 接受来自多个源的数据缓冲区，然后将它们进行合成并发送到显示屏。WindowManager 为 SurfaceFlinger 提供缓冲区和窗口元数据，而 SurfaceFlinger 可使用这些信息将 Surface 合成到屏幕。

SurfaceFlinger 可通过两种方式接受缓冲区：通过 BufferQueue 和 SurfaceControl，或通过 ASurfaceControl。上文中，我们已经介绍了 BufferQueue。ASurfaceControl 是 Android 10 新增的，这是 SurfaceFlinger 接受缓冲区的另一种方式。ASurfaceControl 将 Surface 和 SurfaceControl 组合到一个事务包中，该包会被发送至 SurfaceFlinger。ASurfaceControl 与层相关联，应用可通过 ASurfaceTransactions 更新该层。然后，应用可通过回调（用于传递包含锁定时间、获取时间等信息的 ASurfaceTransactionStats）获取有关 ASurfaceTransactions 的信息。

### 4.2 输入系统

![](<../../.gitbook/assets/image (20).png>)

![](<../../.gitbook/assets/image (205).png>)

Android的用户输入系统获取用户按键（或模拟按键）输入，分发给特定的模块（Framework或应用程序）进行处理，它涉及到以下一些模块：

* **Input Reader:** 负责从硬件获取输入，转换成事件（Event), 并分发给Input Dispatcher.
* **Input Dispatcher:** 将Input Reader传送过来的Events 分发给合适的窗口，并监控ANR。
* **Input Manager Service：** 负责Input Reader 和 Input Dispatchor的创建，并提供Policy 用于Events的预处理。
* **Window Manager Service：**&#x7BA1;理Input Manager 与 View（Window) 以及 ActivityManager 之间的通信。
* **View and Activity：**&#x63A5;收按键并处理。
* **ActivityManager Service：**&#x41;NR 处理。

它们之间的关系如上图所示（黑色箭头代表控制信号传递方向，而红色箭头代表用户输入数据的传递方向）。

## 5. Android渲染的演进

Android 的 UI 渲染性能是 Google 长期以来非常重视的部分（每次 Google I/O 都会花很多篇幅讲这一块）。相比 iOS 系统，Android 的渲染性能一直被人诟病，Android 系统为了弥补跟 iOS 的差距，在每个版本都做了大量的优化。接下来我们通过演进的方式看看 Android 系统在渲染方面都做了哪些努力。

#### **1. Android 4.0 开启硬件加速**&#x20;

在 Android 3.0 之前，或者没有启用硬件加速时，系统都会使用软件方式来渲染 UI。

![](<../../.gitbook/assets/image (322).png>)

整个流程如上图所示：&#x20;

* **Surface:** 每个 View 都由某一个窗口管理，而每一个窗口都会关联有一个 Surface。
* **Canvas:** 通过 Surface 的 lock 函数获得一个 Canvas，Canvas 可以简单理解为 Skia 底层接口的封装。
* **Graphic Buffer:** SurfaceFlinger 会帮助我们托管一个 BufferQueue，我们从 BufferQueue 中拿到Graphic Buffer，然后通过 Canvas 以及 Skia 将绘制内容栅格化到上面。&#x20;
* **SurfaceFlinger：**&#x901A;过 Swap Buffer 把 Front Graphic Buffer 的内容交给 SurfaceFlinger，最后硬件合成器 Hardware Composer 合成并输出到显示屏。

整个渲染流程看上去比较简单，但是正如前面所说，CPU 对于图形处理并不是那么高效，这个过程完全没有利用 GPU 的高性能。

所以从 Android 3.0 开始，Android 开始支持硬件加速，但是到 Android 4.0 时才默认开启硬件加速。

![](<../../.gitbook/assets/image (21).png>)

硬件加速绘制与软件绘制整个流程差异非常大，最核心就是通过 GPU 完成 Graphic Buffer 的内容绘制。此外硬件绘制还引入了一个 DisplayList 的概念，每个 View 内部都有一个 DisplayList，当某个 View 需要重绘时，将它标记为 Dirty。 当需要重绘时，仅仅只需要重绘一个 View 的 DisplayList，而不是像软件绘制那样需要向上递归。这样可以大大减少绘图的操作数量，因而提高了渲染效率。

![](<../../.gitbook/assets/image (232).png>)

#### 2. Android 4.1 Project Butter

&#x20;Google 在 2012 年的 I/O 大会上宣布了 Project Butter 黄油计划，并且在 Android 4.1 中正式开启了这个机制。 Project Butter 主要包含三个组成部分，**VSYNC**、**Triple Buffer** 和 **Choreographer**。&#x20;

**VSYNC 信号:**

VSYNC（Vertical Synchronization）是理解 Project Butter 的核心。对于 Android 4.0，CPU 可能会因为在忙其他的事情，导致没来得及处理 UI 绘制。 为了解决这个问题，Project Butter 引入了 VSYNC，它类似于时钟中断，每收到 VSYNC 中断，CPU 会立即准备 Buffer 数据，由于大部分显示设备刷新频率都是 60 Hz（一秒刷新 60 次），也就是说一帧数据的准备都要在 16ms 内完成。

![](<../../.gitbook/assets/image (321).png>)

这样应用总是在 VSYNC 边界上开始绘制，而 SurfaceFlinger 总是在 VSYNC 边界上进行合成。这样可以消除卡顿，并提升图形的视觉表现。

**Triple Buffering（三缓冲机制）:**

在 Android 4.1 之前，Android 使用双缓冲机制。怎么理解呢？一般来说，不同的 View 或者 Activity 都会共用同一个 Window，也就是共用同一个 Surface。 而每个 Surface 都会有一个 BufferQueue 缓存队列，但是这个队列会由 SurfaceFlinger 管理，通过匿名共享内存机制与 App 应用层交互。

![](<../../.gitbook/assets/image (333).png>)

整个流程如下：&#x20;

* 每个 Surface 对应的 BufferQueue 内部都有两个 Graphic Buffer，一个用于绘制一个用于显示。应用会把内容先绘制到离屏缓冲区（OffScreen Buffer），在需要显示时，才把离屏缓冲区的内容通过 Swap Buffer 复制到 Front Graphic Buffer 中。&#x20;
* 这样 SurfaceFlinge 就拿到了某个 Surface 最终要显示的内容，但是同一时间我们可能会有多个 Surface。这里面可能是不同应用的 Surface，也可能是同一个应用里面类似 SurfaceView 和 TextureView，它们都会有自己单独的 Surface。&#x20;
* 这个时候 SurfaceFlinger 把所有 Surface 要显示的内容统一交给 Hardware Composer，它会根据位置、Z - Order 顺序等信息合成为最终屏幕需要显示的内容，而这个内容交给系统的帧缓冲区 Frame Buffer 来显示（Frame Buffer 是非常底层的，可以理解为屏幕显示的抽象）。

如果你理解了双缓冲机制的原理，那就非常容易理解什么是三缓冲区了。如果只有两个 Graphic Buffer 缓存区 A 和 B，如果 CPU / GPU 绘制过程较长，超过了一个 VSYNC 信号周期，因为缓冲区 B 中的数据还没有准备完成，所以只能继续展示 A 缓冲区的内容，这样缓冲区 A 和 B 都分别被显示设备和 GPU 占用，CPU 无法准备下一帧的数据。

![](<../../.gitbook/assets/image (346).png>)

如果再提供一个缓冲区，CPU、GPU 和显示设备都能使用各自的缓冲区工作，互不影响。简单来说，三缓冲机制就是在双缓冲机制的基础上增加了一个 Graphic Buffer 缓冲区，这样可以最大限度的利用空闲时间，带来的坏处是多使用了一个 Graphic Buffer 所占用的内存。

![](<../../.gitbook/assets/image (326).png>)

**Choreographer：**

Choreographer 本质是一个 Java 类，如果直译的话为舞蹈指导，看到这个词不得不赞叹设计者除了 Coding 之外的广泛视野。舞蹈是有节奏的，节奏使舞蹈的每个动作更加协调和连贯；视图刷新也是如此，Choreographer 可以接收系统的 VSYNC 信号，业界一般通过它来监控应用的帧率。 Choreographer 是线程单例，而且它具有处理当前线程 Looper 的能力。

#### 3. Android 5.0：RenderThread&#x20;

经过 Android 4.1 的 Project Butter 黄油计划之后，Android 的渲染性能有了很大的改善。不过你有没有注意到这样一个问题，虽然利用了 GPU 的图形高性能运算，但是从计算 DisplayList，到通过 GPU 绘制到 Frame Buffer，整个计算和绘制都在 UI 主线程中完成。

![](<../../.gitbook/assets/image (100).png>)

UI 线程“既当爹又当妈”，任务过于繁重。如果整个渲染过程比较耗时，可能造成无法响应用户的操作，进而出现卡顿的情况。GPU 对图形的绘制渲染能力更胜一筹，如果**使用 GPU 并在不同线程绘制渲染图形**，那么整个流程会更加顺畅。&#x20;

正因如此，在 Android 5.0 引入两个比较大的改变。一个是引入了 **RenderNode** 的概念，它对 DisplayList 及一些 View 显示属性都做了进一步封装。另一个是引入了 **RenderThread**，所有的 GL 命令执行都放到这个线程上，渲染线程在 RenderNode 中存有渲染帧的所有信息，可以做一些属性动画，这样即便主线程有耗时操作的时候也可以保证动画流程。

## 6. 参考资料

* [Drawn out: How Android renders (Google I/O '18)](https://www.youtube.com/watch?v=zdQRIYOST64\&ab_channel=AndroidDevelopers)





