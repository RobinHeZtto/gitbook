# Android基础之显示机制全解析

##

> Android 的演进中，引入了 **Vsync + TripleBuffer + Choreographer** 的机制，其主要目的就是提供一个稳定的帧率输出机制，让软件层和硬件层可以以共同的频率一起工作。

## [https://source.android.com/devices/graphics?hl=zh-cn](https://source.android.com/devices/graphics?hl=zh-cn)

![](<../../.gitbook/assets/image (404).png>)

## > 绘制 -> 合成 -> 显示



![](<../../.gitbook/assets/image (332).png>)



## 1. 显示系统基础知识

在一个典型的显示系统中，一般包括CPU、GPU、Display三个部分， CPU负责计算帧数据，把计算好的数据交给GPU，GPU会对图形数据进行渲染，渲染好后放到buffer(图像缓冲区)里存起来，然后Display（屏幕或显示器）负责把buffer里的数据呈现到屏幕上。如下图：\


![](<../../.gitbook/assets/image (67).png>)

### 1.1 基础概念

* **屏幕刷新频率**

一秒内屏幕刷新的次数（一秒内显示了多少帧的图像），单位 Hz（赫兹），如常见的 60 Hz。**刷新频率取决于硬件的固定参数**（不会变的）。

* **逐行扫描**

显示器并不是一次性将画面显示到屏幕上，而是从左到右边，从上到下逐行扫描，顺序显示整屏的一个个像素点，不过这一过程快到人眼无法察觉到变化。以 60 Hz 刷新率的屏幕为例，这一过程即 1000 / 60 ≈ 16ms。

* **帧率** （Frame Rate）

表示 **GPU 在一秒内绘制操作的帧数**，单位 fps。例如在电影界采用 24 帧的速度足够使画面运行的非常流畅。而 Android 系统则采用更加流程的 60 fps，即每秒钟GPU最多绘制 60 帧画面。帧率是动态变化的，例如当画面静止时，GPU 是没有绘制操作的，屏幕刷新的还是buffer中的数据，即GPU最后操作的帧数据。

* **画面撕裂**（tearing）

一个屏幕内的数据来自2个不同的帧，画面会出现撕裂感，如下图

![](<../../.gitbook/assets/image (341).png>)

### 2.2 双缓存

#### 2.2.1 画面撕裂 原因

屏幕刷新频是固定的，比如每16.6ms从buffer取数据显示完一帧，理想情况下帧率和刷新频率保持一致，即每绘制完成一帧，显示器显示一帧。但是CPU/GPU写数据是不可控的，所以会出现buffer里有些数据根本没显示出来就被重写了，即buffer里的数据可能是来自不同的帧的， 当屏幕刷新时，此时它并不知道buffer的状态，因此从buffer抓取的帧并不是完整的一帧画面，即出现画面撕裂。

简单说就是Display在显示的过程中，buffer内数据被CPU/GPU修改，导致画面撕裂。

#### 2.2.2 双缓存

那咋解决画面撕裂呢？ 答案是使用 双缓存。

由于图像绘制和屏幕读取 使用的是同个buffer，所以屏幕刷新时可能读取到的是不完整的一帧画面。

**双缓存**，让绘制和显示器拥有各自的buffer：GPU 始终将完成的一帧图像数据写入到 **Back Buffer**，而显示器使用 **Frame Buffer**，当屏幕刷新时，Frame Buffer 并不会发生变化，当Back buffer准备就绪后，它们才进行交换。如下图：

![双缓存，CPU/GPU写数据到Back Buffer，显示器从Frame Buffer取数据](https://img-blog.csdnimg.cn/202008192024268.png#pic_center)

#### 2.2.3 VSync

问题又来了：什么时候进行两个buffer的交换呢？

假如是 Back buffer准备完成一帧数据以后就进行，那么如果此时屏幕还没有完整显示上一帧内容的话，肯定是会出问题的。看来只能是等到屏幕处理完一帧数据后，才可以执行这一操作了。

当扫描完一个屏幕后，设备需要重新回到第一行以进入下一次的循环，此时有一段时间空隙，称为VerticalBlanking Interval(VBI)。那，这个时间点就是我们进行缓冲区交换的最佳时间。因为此时屏幕没有在刷新，也就避免了交换过程中出现 screen tearing的状况。

**VSync**(垂直同步)是VerticalSynchronization的简写，它利用VBI时期出现的vertical sync pulse（垂直同步脉冲）来保证双缓冲在最佳时间点才进行交换。另外，交换是指各自的内存地址，可以认为该操作是瞬间完成。

所以说V-sync这个概念并不是Google首创的，它在早年的PC机领域就已经出现了。

## 2. Android屏幕刷新机制

### 3.1 Android4.1之前的问题

具体到Android中，在Android4.1之前，屏幕刷新也遵循 上面介绍的 双缓存+VSync 机制。如下图：&#x20;

![双缓存会在VSync脉冲时交换，但CPU/GPU绘制是随机的](https://img-blog.csdnimg.cn/20200819205135422.png#pic_center)

以时间的顺序来看下将会发生的过程：

1. Display显示第0帧数据，此时CPU和GPU渲染第1帧画面，且在Display显示下一帧前完成
2. 因为渲染及时，Display在第0帧显示完成后，也就是第1个VSync后，缓存进行交换，然后正常显示第1帧
3. 接着第2帧开始处理，是直到第2个VSync快来前才开始处理的。
4. 第2个VSync来时，由于第2帧数据还没有准备就绪，缓存没有交换，显示的还是第1帧。这种情况被Android开发组命名为“Jank”，即发生了**丢帧**。
5. 当第2帧数据准备完成后，它并不会马上被显示，而是要等待下一个VSync 进行缓存交换再显示。

所以总的来说，就是屏幕平白无故地多显示了一次第1帧。原因是第2帧的CPU/GPU计算没能在VSync信号到来前完成 。

我们知道，**双缓存的交换是在Vsyn到来时进行，交换后屏幕会取Frame buffer内的新数据，而实际此时的Back buffer 就可以供GPU准备下一帧数据了。 如果 Vsync到来时 CPU/GPU就开始操作的话，是有完整的16.6ms的，这样应该会基本避免jank的出现了**（除非CPU/GPU计算超过了16.6ms）。 那如何让 CPU/GPU计算在 Vsync到来时进行呢？

### 3.2 drawing with VSync

为了优化显示性能，Google在Android 4.1系统中对Android Display系统进行了重构，实现了Project Butter（黄油工程）：系统在收到VSync pulse后，将马上开始下一帧的渲染。即**一旦收到VSync通知（16ms触发一次），CPU和GPU 才立刻开始计算然后把数据写入buffer**。如下图：  CPU/GPU根据VSYNC信号同步处理数据，可以让CPU/GPU有完整的16ms时间来处理数据，减少了jank。

![VSync脉冲到来：双缓存交换，且开始CPU/GPU绘制](https://img-blog.csdnimg.cn/20200819212951194.png#pic_center)

一句话总结，**VSync同步使得CPU/GPU充分利用了16.6ms时间，减少jank。**

问题又来了，如果界面比较复杂，CPU/GPU的处理时间较长 超过了16.6ms呢？如下图：&#x20;

![虽然CPU/GPU开始在VSync，但超过16.6ms](https://img-blog.csdnimg.cn/2020081921343523.png#pic_center)

1. 在第二个时间段内，但却因 GPU 还在处理 B 帧，缓存没能交换，导致 A 帧被重复显示。
2. 而B完成后，又因为缺乏VSync pulse信号，它只能等待下一个signal的来临。于是在这一过程中，有一大段时间是被浪费的。
3. 当下一个VSync出现时，CPU/GPU马上执行操作（A帧），且缓存交换，相应的显示屏对应的就是B。这时看起来就是正常的。只不过由于执行时间仍然超过16ms，导致下一次应该执行的缓冲区交换又被推迟了——如此循环反复，便出现了越来越多的“Jank”。

**为什么 CPU 不能在第二个 16ms 处理绘制工作呢？**

原因是只有两个 buffer，Back buffer正在被GPU用来处理B帧的数据， Frame buffer的内容用于Display的显示，这样两个buffer都被占用，CPU 则无法准备下一帧的数据。 那么，如果再提供一个buffer，CPU、GPU 和显示设备都能使用各自的buffer工作，互不影响。

### 3.3 三缓存

**三缓存**就是在双缓冲机制基础上增加了一个 Graphic Buffer 缓冲区，这样可以最大限度的利用空闲时间，带来的坏处是多使用的一个 Graphic Buffer 所占用的内存。&#x20;

![三缓存](https://img-blog.csdnimg.cn/20200819220105382.png#pic_center)

1. 第一个Jank，是不可避免的。但是在第二个 16ms 时间段，CPU/GPU 使用 **第三个 Buffer** 完成C帧的计算，虽然还是会多显示一次 A 帧，但后续显示就比较顺畅了，有效避免 Jank 的进一步加剧。
2. 注意在第3段中，A帧的计算已完成，但是在第4个vsync来的时候才显示，如果是双缓冲，那在第三个vynsc就可以显示了。

**三缓冲有效利用了等待vysnc的时间，减少了jank，但是带来了延迟。** 所以，是不是 Buffer 越多越好呢？这个是否定的，Buffer 正常还是两个，当出现 Jank 后三个足以。

以上就是Android屏幕刷新的原理了。



## 1. 基础概念

应用开发者可通过两种方式将图像绘制到屏幕上：使用 `Canvas` 或 `OpenGL` ：

* `android.graphics.Canvas` 是一个 `2D` 图形 `API` ， `Canvas API` 通过一个名为 `OpenGLRenderer` 的绘制库实现硬件加速，该绘制库将 `Canvas` 运算转换为 `OpenGL` 运算，以便它们可以在 `GPU` 上执行。从 `Android 4.0` 开始，硬件加速的 `Canvas` 默认情况下处于启用状态
* 除了 `Canvas`，开发者渲染图形的另一个主要方式是使用 `OpenGL ES` 直接渲染到 `Surface` 。 `Android` 在 `Android.opengl` 软件包中提供了 `OpenGL ES` 接口

#### EGL

先熟悉 `Android` 平台图形处理 `API` 的标准：

* `OpenGL`\
  是由 `SGI` 公司开发的一套 `3D` 图形软件接口标准，由于具有体系结构简单合理、使用方便、与操作平台无关等优点， `OpenGL` 迅速成为 `3D` 图形接口的工业标准，并陆续在各种平台上得以实现。
* `OpenGL ES`\
  是由 `khronos` 组织根据手持及移动平台的特点，对 `OpenGL 3D` 图形 `API` 标准进行裁剪定制而形成的。
* `Vulkan`\
  是由 `khronos` 组织在 2016 年正式发布的，是 `OpenGL ES` 的继任者。 `API` 是轻量级、更贴近底层硬件 `close-to-the-metal` 的接口，可使 `GPU` 驱动软件运用多核与多线程 `CPU` 性能。

`OpenGL ES` 定义了一个渲染图形的 `API` ，但没有定义窗口系统。为了让它能够适合各种平台，它将与知道如何通过操作系统创建和访问窗口的库结合使用。而在 `Android` 中，这个库被称为 `EGL` ；也就是说 `EGL` 主要是适配系统和关联窗口属性。如果要绘制纹理多边形，应使用 `OpenGL ES` 调用；如果要在屏幕上进行渲染，应使用 `EGL` 调用。\
`OpenGL ES` 是 `Android` 绘图 `API` ，但 `OpenGL ES` 是平台通用的，在特定设备上使用需要一个中间层做适配， `Android` 中这个中间层就是 `EGL` 。

[![0113-android-graphics-display-android-egl.jpg](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-android-egl.jpg)](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-android-egl.jpg)

#### Surface和SurfaceFlinger

无论开发者使用什么渲染 `API`，一切内容都会渲染到 `Surface` 。 `Surface` 表示缓冲队列中的生产者，而缓冲队列通常会被 `SurfaceFlinger` 消耗。在 `Android` 平台上创建的每个窗口都由 `Surface` 提供支持。所有被渲染的可见 `Surface` 都被 `SurfaceFlinger` 合成到显示部分。它们遵循生产者/消费者模型：

#### WindowManagerService

窗口管理器，控制窗口的 `Android` 系统服务，它是视图容器。窗口总是由 `Surface` 提供支持。该服务会监督生命周期、输入和聚焦事件、屏幕方向、转换、动画、位置、变形、 `Z-Order` 以及窗口的其他许多方面。窗口管理器会将所有窗口元数据发送到 `SurfaceFlinger` ，以便 `SurfaceFlinger` 可以使用该数据在显示部分合成 `Surface` 。

#### Android 图形组件

无论开发者使用什么渲染 API，一切内容都会渲染&#x5230;**“Surface”**。**Surface** 表示缓冲队列中的生产方，而缓冲队列通常会被 SurfaceFlinger 消耗。在 Android 平台上创建的每个窗口都由 Surface 提供支持。所有被渲染的可见 Surface 都被 SurfaceFlinger 合成到显示部分。

![](<../../.gitbook/assets/image (77).png>)

#### 图像流生产方 <a href="#image_stream_producers" id="image_stream_producers"></a>

图像流生产方可以是生成图形缓冲区以供消耗的任何内容。例如 OpenGL ES、Canvas 2D 和 mediaserver 视频解码器。

#### 图像流消耗方 <a href="#image_stream_consumers" id="image_stream_consumers"></a>

图像流的最常见消耗方是 SurfaceFlinger，该系统服务会消耗当前可见的 Surface，并使用窗口管理器中提供的信息将它们合成到显示部分。SurfaceFlinger 是可以修改所显示部分内容的唯一服务。SurfaceFlinger 使用 OpenGL 和 Hardware Composer 来合成一组 Surface。

其他 OpenGL ES 应用也可以消耗图像流，例如相机应用会消耗相机预览图像流。非 GL 应用也可以是使用方，例如 ImageReader 类。

#### 硬件混合渲染器 <a href="#hardware_composer" id="hardware_composer"></a>

显示子系统的硬件抽象实现。SurfaceFlinger 可以将某些合成工作委托给 Hardware Composer，以分担 OpenGL 和 GPU 上的工作量。SurfaceFlinger 只是充当另一个 OpenGL ES 客户端。因此，在 SurfaceFlinger 将一个或两个缓冲区合成到第三个缓冲区中的过程中，它会使用 OpenGL ES。这样使合成的功耗比通过 GPU 执行所有计算更低。

[Hardware Composer HAL](https://source.android.com/devices/graphics/architecture#hwcomposer) 则进行另一半的工作，并且是所有 Android 图形渲染的核心。Hardware Composer 必须支持事件，其中之一是 VSYNC.



#### 数据流

![](<../../.gitbook/assets/image (102).png>)

左侧的对象是生成图形缓冲区的渲染器，如主屏幕、状态栏和系统界面。SurfaceFlinger 是合成器，而硬件混合渲染器是制作器。

![](<../../.gitbook/assets/image (215).png>)





## 4. SurfaceFlinger

官方注释: `SurfaceFlinger accepts buffers, composes buffers, and sends buffers to the display.The most common consumer of image streams is SurfaceFlinger, the system service that consumes the currently visible surfaces and composites them onto the display using information provided by the Window Manager. SurfaceFlinger is the only service that can modify the content of the display. SurfaceFlinger uses OpenGL and the Hardware Composer to compose a group of surfaces.`\
SurfaceFlinger 用来管理消费当前可见的 Surface, 所有被渲染的可见 Surface 都会被 SurfaceFlinger 通过 WindowManager 提供的信息合成(使用 OpenGL 和 HardWare Composer)提交到屏幕的后缓冲区，等待屏幕的下一个 Vsync 信号到来，再显示到屏幕上。SufaceFlinger 通过屏幕后缓冲区与屏幕建立联系，同时通过 Surface 与上层建立联系，起到了一个承上启下的作用。

**SurfaceFlinger与WindowManager的关系：**

SurfaceFlinger 接受缓冲区，对它们进行合成，然后发送到屏幕。WindowManager 为 SurfaceFlinger 提供缓冲区和窗口元数据，而 SurfaceFlinger 可使用这些信息将 Surface 合成到屏幕。

SurfaceFlinger 可通过两种方式接受缓冲区：通过 BufferQueue 和 SurfaceControl，或通过 ASurfaceControl。

SurfaceFlinger 接受缓冲区的一种方式是通过 BufferQueue 和 SurfaceControl。当应用进入前台时，它会从 WindowManager 请求缓冲区。然后，WindowManager 会从 SurfaceFlinger 请求层。层是 [surface](https://source.android.com/devices/graphics/arch-sh?hl=zh-cn)（包含 BufferQueue）和 [SurfaceControl](https://developer.android.com/reference/android/view/SurfaceControl?hl=zh-cn)（包含屏幕框架等层元数据）的组合。SurfaceFlinger 创建层并将其发送至 WindowManager。然后，WindowManager 将 Surface 发送至应用，但会保留 SurfaceControl 来操控应用在屏幕上的外观。

### 启动SurfaceFlinger

在解析执行了 init.rc 文件后，surfaceflinger 服务启动从 main\_surfaceflinger.main() 方法开始：

```
// frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp
int main(int, char**) {
    // When SF is launched in its own process, limit the number of binder threads to 4.
    ProcessState::self()->setThreadPoolMaxThreadCount(4);

    // start the thread pool
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    // instantiate surfaceflinger
    sp<SurfaceFlinger> flinger = new SurfaceFlinger();
    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    set_sched_policy(0, SP_FOREGROUND);
    // Put most SurfaceFlinger threads in the system-background cpuset Keeps us from unnecessarily
    // using big cores. Do this after the binder thread pool init
    if (cpusets_enabled()) set_cpuset_policy(0, SP_SYSTEM);

    // initialize before clients can connect
    flinger->init();

    // 将 “SurfaceFlinger” Binder 服务注册到 ServiceManager
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false, IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL);

    // 将 “gpu” Binder 服务注册到 ServiceManager
    sp<GpuService> gpuservice = new GpuService();
    sm->addService(String16(GpuService::SERVICE_NAME), gpuservice, false);

    startDisplayService(); // dependency on SF getting registered above

    // run surface flinger in this thread
    flinger->run();

    return 0;
}
```

从注释可以看出，该方法的主要功能如下：

* 设置 surfaceflinger 进程的 binder 线程池数最多为4，然后启动 binder 线程池；
* 创建 SurfaceFlinger 对象，将 surfaceflinger 进程设置为高优先级及使用前台调度策略；
* 初始化 SurfaceFlinger；
* 将 `SurfaceFlinger` 和 `gpu` Binder 服务注册到 ServiceManager；
* 启动 DisplayService；
* 在当前主线程执行 SurfaceFlinger.run 方法。

### 实例化SurfaceFlinger

```
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

SurfaceFlinger::SurfaceFlinger() : SurfaceFlinger(SkipInitialization) {
    // ...
}

// 执行这里
void SurfaceFlinger::onFirstRef() {
    // MessageQueue 类型
    mEventQueue->init(this);
}

// 消息机制
void MessageQueue::init(const sp<SurfaceFlinger>& flinger) {
    mFlinger = flinger;
    mLooper = new Looper(true);
    mHandler = new Handler(*this);
}
```

### 初始化SurfaceFlinger

SF 初始化的逻辑在 SurfaceFlinger.init 方法里：

```
void SurfaceFlinger::init() {
    ALOGI("SurfaceFlinger's main thread ready to run. " "Initializing graphics H/W...");

    // 分别创建 app 和 SF 的 EventThread 线程
    mEventThreadSource = std::make_unique<DispSyncSource>(&mPrimaryDispSync, SurfaceFlinger::vsyncPhaseOffsetNs, true, "app");
    mEventThread = std::make_unique<impl::EventThread>(mEventThreadSource.get(), [this]() { resyncWithRateLimit(); },
            impl::EventThread::InterceptVSyncsCallback(), "appEventThread");
    mSfEventThreadSource = std::make_unique<DispSyncSource>(&mPrimaryDispSync, SurfaceFlinger::sfVsyncPhaseOffsetNs, true, "sf");
    mSFEventThread = std::make_unique<impl::EventThread>(mSfEventThreadSource.get(), [this]() { resyncWithRateLimit(); },
            [this](nsecs_t timestamp) { mInterceptor->saveVSyncEvent(timestamp); }, "sfEventThread");
    mEventQueue->setEventThread(mSFEventThread.get());
    mVsyncModulator.setEventThread(mSFEventThread.get());

    // 获取 RenderEngine 引擎(can't fail)
    getBE().mRenderEngine = RE::impl::RenderEngine::create(HAL_PIXEL_FORMAT_RGBA_8888, hasWideColorDisplay ? RE::RenderEngine::WIDE_COLOR_SUPPORT : 0);
    // 初始化 Hardware Composer 对象(通过 HAL 层的 HWComposer 硬件模块 或 软件模拟产生 Vsync 信号)
    getBE().mHwc.reset(new HWComposer(std::make_unique<Hwc2::impl::Composer>(getBE().mHwcServiceName)));
    getBE().mHwc->registerCallback(this, getBE().mComposerSequenceId);
    // 初始化显示屏 DisplayDevice
    processDisplayHotplugEventsLocked();

    // make the default display GLContext current so that we can create textures
    // when creating Layers (which may happens before we render something)
    getDefaultDisplayDeviceLocked()->makeCurrent();

    mEventControlThread = std::make_unique<impl::EventControlThread>(
            [this](bool enabled) { setVsyncEnabled(HWC_DISPLAY_PRIMARY, enabled); });

    // initialize our drawing state
    mDrawingState = mCurrentState;
    // set initial conditions (e.g. unblank default device)
    initializeDisplays();

    // 启动开机动画服务
    if (mStartPropertySetThread->Start() != NO_ERROR) {
        ALOGE("Run StartPropertySetThread failed!");
    }
}
```

SF 初始化时的主要功能：

* 创建 HWComposer 对象；
* 初始化非虚拟显示屏；
* 分别启动 app 和 sf 的 EventThread 线程，创建了两个 DispSyncSource 对象，分别是用于绘制(app)和合成(SurfaceFlinger)；
* 启动开机动画服务；

### 向HWC注册回调

在前一节的 SurfaceFlinger.init 方法中有一行代码: `getBE().mHwc->registerCallback(this, getBE().mComposerSequenceId)` 给 HWComposer 注册了回调：

```
// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
void HWComposer::registerCallback(HWC2::ComposerCallback* callback, int32_t sequenceId) {
    // mHwcDevice 是 Device 类型
    mHwcDevice->registerCallback(callback, sequenceId);
}

void Device::registerCallback(ComposerCallback* callback, int32_t sequenceId) {
    if (mRegisteredCallback) {
        ALOGW("Callback already registered. Ignored extra registration attempt.");
        return;
    }
    mRegisteredCallback = true;
    sp<ComposerCallbackBridge> callbackBridge(new ComposerCallbackBridge(callback, sequenceId));
    mComposer->registerCallback(callbackBridge);
}

// 通过 mClient 向硬件注册回调
void Composer::registerCallback(const sp<IComposerCallback>& callback) {
    auto ret = mClient->registerCallback(callback);
}
```

然后可以看一下 ComposerCallbackBridge 回调接口：

```
class ComposerCallbackBridge : public Hwc2::IComposerCallback {
    // 热插拔
    Return<void> onHotplug(Hwc2::Display display, IComposerCallback::Connection conn) override {
        HWC2::Connection connection = static_cast<HWC2::Connection>(conn);
        mCallback->onHotplugReceived(mSequenceId, display, connection);
        return Void();
    }

    // 刷新
    Return<void> onRefresh(Hwc2::Display display) override {
        mCallback->onRefreshReceived(mSequenceId, display);
        return Void();
    }

    // Vsync
    Return<void> onVsync(Hwc2::Display display, int64_t timestamp) override {
        mCallback->onVsyncReceived(mSequenceId, display, timestamp);
        return Void();
    }
};
```

上面 ComposerCallbackBridge 中的 mCallback 参数，传入的是 SurfaceFlinger 实例，它继承自 `HWC2::ComposerCallback` 接口，用来处理 HWC 的回调。

### 接收HWC回调

当 HWComposer 产生 Vsync 信号时，回调 ComposerCallbackBridge.onVsync 方法，进而调用 SF.onVsyncReceived 方法：

```
void SurfaceFlinger::onVsyncReceived(int32_t sequenceId, hwc2_display_t displayId, int64_t timestamp) {
    // ...
    bool needsHwVsync = false;
    {
        Mutex::Autolock _l(mHWVsyncLock);
        if (type == DisplayDevice::DISPLAY_PRIMARY && mPrimaryHWVsyncEnabled) {
            // mPrimaryHWVsyncEnabled 标识主屏幕对应的 HWC 的 VSYNC 功能有没有被开启
            needsHwVsync = mPrimaryDispSync.addResyncSample(timestamp);
        }
    }
    if (needsHwVsync) {
        enableHardwareVsync();
    } else {
        disableHardwareVsync(false);
    }
}
```

mPrimaryDispSync 是一个 DispSync 对象，可以看出硬件产生的 Vsync 信号会交给同步模型 DispSync 的 addResyncSample 方法处理，根据该方法的返回值可以控制硬件是否继续发送垂直信号(SF.enableHardwareVsync & SF.disableHardwareVsync)，即硬件的垂直信号并不是持续产生的，而是 DispSync 同步模型在需要(addResyncSample)的时候才 enable 的。

```
void SurfaceFlinger::enableHardwareVsync() {
    if (!mPrimaryHWVsyncEnabled && mHWVsyncAvailable) {
        mPrimaryDispSync.beginResync();
        mEventControlThread->setVsyncEnabled(true);
        mPrimaryHWVsyncEnabled = true;
    }
}

void SurfaceFlinger::disableHardwareVsync(bool makeUnavailable) {
    Mutex::Autolock _l(mHWVsyncLock);
    if (mPrimaryHWVsyncEnabled) {
        mEventControlThread->setVsyncEnabled(false);
        mPrimaryDispSync.endResync();
        mPrimaryHWVsyncEnabled = false;
    }
    if (makeUnavailable) {
        mHWVsyncAvailable = false;
    }
}
```

可以看到是通过 EventControlThread 来完成的。

### EventControlThread

EventControlThread 线程用来控制 HWC 是否应该发送 Vsync 信号。

```
EventControlThread::EventControlThread(EventControlThread::SetVSyncEnabledFunction function) : mSetVSyncEnabled(function) {
    pthread_setname_np(mThread.native_handle(), "EventControlThread");
    // ...
}

void EventControlThread::setVsyncEnabled(bool enabled) {
    std::lock_guard<std::mutex> lock(mMutex);
    mVsyncEnabled = enabled;
    mCondition.notify_all();
}

void EventControlThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    auto keepRunning = true;
    auto currentVsyncEnabled = false;

    while (keepRunning) {
        mSetVSyncEnabled(currentVsyncEnabled); // false--初始化 SF 时默认会关闭 HWC Vsync 的发送

        std::unique_lock<std::mutex> lock(mMutex);
        mCondition.wait(lock, [this, currentVsyncEnabled, keepRunning]() NO_THREAD_SAFETY_ANALYSIS {
            return currentVsyncEnabled != mVsyncEnabled || keepRunning != mKeepRunning;
        });
        currentVsyncEnabled = mVsyncEnabled;
        keepRunning = mKeepRunning;
    }
}

// 根据之前的代码可以知道 mSetVSyncEnabled 传入的是 SF.setVsyncEnabled
void SurfaceFlinger::setVsyncEnabled(int disp, int enabled) {
    // 控制 HWC 是否使能 Vsync
    getHwComposer().setVsyncEnabled(disp, enabled ? HWC2::Vsync::Enable : HWC2::Vsync::Disable);
}
```

可以看到默认会关闭 HWC Vsync 的发送，然后在执行 initializeDisplays 时会将其打开，调用流程: `SF.initializeDisplays -> SF.onInitializeDisplays -> SF.setPowerModeInternal -> SF.resyncToHardwareVsync(true)`

```
void SurfaceFlinger::resyncToHardwareVsync(bool makeAvailable) {
    Mutex::Autolock _l(mHWVsyncLock);

    if (makeAvailable) {
        mHWVsyncAvailable = true;
    } else if (!mHWVsyncAvailable) {
        // Hardware vsync is not currently available, so abort the resync
        // attempt for now
        return;
    }

    const auto& activeConfig = getBE().mHwc->getActiveConfig(HWC_DISPLAY_PRIMARY);
    const nsecs_t period = activeConfig->getVsyncPeriod();

    mPrimaryDispSync.reset();
    mPrimaryDispSync.setPeriod(period);

    if (!mPrimaryHWVsyncEnabled) {
        mPrimaryDispSync.beginResync();
        mEventControlThread->setVsyncEnabled(true);
        mPrimaryHWVsyncEnabled = true;
    }
}
```

### DispSync同步模型

在 SF 的初始化过程中创建了两个 DispSyncSource 对象，分别是 App 绘制延时源和 SF 合成延时源，这两个源都基于 mPrimaryDispSync---一个 DispSync 对象，是对硬件 Hwc 垂直信号的同步模型，它在 SF 的构造方法中实例化。**这是 Android 的优化策略，因为在 Vsync 信号到来后如果同时进行 App 的绘制和 SF 的合成流程，则可能会竞争 CPU 资源，从而影响效率。因此引入了 Vsync 同步模型 DispSync, 该模型会根据需要打开硬件的 Vsync 信号进行采样，然后同步 DispSync 模型，从而为上层的 APP 绘制延时源和 SF 合成延时源提供 Vsync 信号，这两个延时源分别添加一个相位偏移量(即上面代码中的 vsyncPhaseOffsetNs 和 sfVsyncPhaseOffsetNs)，以此错开在 Vsync 信号到来后 APP 绘制和 SF 合成的执行。**

看一下 DispSync 的初始化：

```
DispSync::DispSync(const char* name) : mName(name), mRefreshSkipCount(0), mThread(new DispSyncThread(name)) {}
```

在 SurfaceFlinger 的构造函数里，会调用 mPrimaryDispSync.init 方法：

```
void DispSync::init(bool hasSyncFramework, int64_t dispSyncPresentTimeOffset) {
    // 启动名为 DispSync 的线程
    mThread->run("DispSync", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);
    reset();
    beginResync();
    // ...
}

class DispSyncThread : public Thread {
    virtual bool threadLoop() {
        while (true) {
            // 调用 mCond.wait(mMutex) 等待被唤醒
            // 其他逻辑...
        }
        return false;
    }
}
```

在执行 DispSync 的初始化方法时会启动名为 DispSync 的线程, DispSync 线程通过调用 mCond.wait(mMutex) 阻塞自身线程，等待被唤醒。在接收到硬件 Vsync 信号后会回调到 SF.onVsyncReceived 方法，其中调用的 DispSync.addResyncSample 方法会更新同步模型的偏移量来使其和硬件的 VSYNC 信号同步，然后再通过 mCond.signal 唤醒 DispSync 线程。具体的算法就不看了，直接看 DispSync 线程被唤醒后的逻辑：

```
virtual bool threadLoop() {
    while (true) {
        // 调用 mCond.wait(mMutex) 等待被唤醒
        // 计算下一次 VSYNC 信号的时间
        targetTime = computeNextEventTimeLocked(now);
        // 还未到触发时间，则等待一段时间
        if (now < targetTime) {
            // wait
        }
        // 收集此次应该通知 Vsync 信号的所有监听者
        Vector<CallbackInvocation> callbackInvocations = gatherCallbackInvocationsLocked(now);
        if (callbackInvocations.size() > 0) {
            fireCallbackInvocations(callbackInvocations);
        }
    }
    return false;
}

Vector<CallbackInvocation> gatherCallbackInvocationsLocked(nsecs_t now) {
    Vector<CallbackInvocation> callbackInvocations;
    nsecs_t onePeriodAgo = now - mPeriod;

    for (size_t i = 0; i < mEventListeners.size(); i++) {
        // 每个监听者下一次 VSYNC 信号的发生时间可能都不同，因为可能设置了不同的偏移
        // 因此针对每个监听者都要计算下一次VSYNC信号
        nsecs_t t = computeListenerNextEventTimeLocked(mEventListeners[i], onePeriodAgo);
        if (t < now) {
            // 需要通知则添加
            CallbackInvocation ci;
            ci.mCallback = mEventListeners[i].mCallback;
            ci.mEventTime = t;
            callbackInvocations.push(ci);
            mEventListeners.editItemAt(i).mLastEventTime = t;
        }
    }
    return callbackInvocations;
}

void fireCallbackInvocations(const Vector<CallbackInvocation>& callbacks) {
    for (size_t i = 0; i < callbacks.size(); i++) {
        // 回调所有的监听者的 onDispSyncEvent 方法
        callbacks[i].mCallback->onDispSyncEvent(callbacks[i].mEventTime);
    }
}

status_t DispSync::addEventListener(const char* name, nsecs_t phase, Callback* callback) {
    // mThread 是 DispSyncThread 类型，朝 mEventListeners 中添加监听器
    return mThread->addEventListener(name, phase, callback);
}
```

上面 DispSync.addEventListener 方法会添加 DispSync 同步模型的 Vsync 信号的监听器。

### DispSyncSource延时源

上面已经讲过在 SF 的初始化过程中创建了两个 DispSyncSource 对象，分别是 App 绘制延时源和 SF 合成延时源，这两个源都基于 DispSync 同步模型来处理硬件 Vsync 信号，延时源通过对应的 EventThread 来管理。下面根据 DispSyncSource 源码看看从 DispSync 同步模型传递给 DispSyncSource 延时源的 Vsync 信号是怎么传递给需要的监听者的：

```
class VSyncSource {
    class Callback {
        virtual void onVSyncEvent(nsecs_t when) = 0;
    };

    virtual void setVSyncEnabled(bool enable) = 0;
    virtual void setCallback(Callback* callback) = 0;
    virtual void setPhaseOffset(nsecs_t phaseOffset) = 0;
};

class DispSync {
    class Callback {
        virtual void onDispSyncEvent(nsecs_t when) = 0;
    };
}

class DispSyncSource final : public VSyncSource, private DispSync::Callback {
    DispSyncSource(DispSync* dispSync, nsecs_t phaseOffset, bool traceVsync, const char* name) :
        mName(name), mDispSync(dispSync), mPhaseOffset(phaseOffset), mEnabled(false) {}

    void setVSyncEnabled(bool enable) override {
        if (enable) { // 向 DispSync 添加监听器
            mDispSync->addEventListener(mName, mPhaseOffset, static_cast<DispSync::Callback*>(this));
        } else { // 移除对 DispSync 的监听器
            mDispSync->removeEventListener(static_cast<DispSync::Callback*>(this));
        }
        mEnabled = enable;
    }

    void setCallback(VSyncSource::Callback* callback) override{
        mCallback = callback;
    }

    // 收到 Vsync 信号后回调
    virtual void onDispSyncEvent(nsecs_t when) {
        VSyncSource::Callback* callback;
        {
            Mutex::Autolock lock(mCallbackMutex);
            callback = mCallback;
        }
        if (callback != nullptr) {
            callback->onVSyncEvent(when);
        }
    }
}
```

DispSyncSource 延时源通过 DispSync 同步模型来构造实例，DispSync 通过调用接口方法 onDispSyncEvent 来通知 DispSyncSource 延时源收到了 Vsync 信号，然后通过 DispSyncSource 延时源设置的 callback.onVSyncEvent 方法将 Vsync 信号的到达事件通知给监听者(其实就是EventThread)。

* setVSyncEnabled: 控制是否监听来自于 DispSync 同步模型的 Vsync 信号
* setCallback: 设置 DispSyncSource 延时源收到来自 DispSync 同步模型的 Vsync 信号后的监听者
* onDispSyncEvent: DispSync 同步模型收到 Vsync 信号后通知 DispSyncSource 的回调方法，方法内部会回调通知通过 setCallback 方法设置的监听者(EventThread)

### EventThread

EventThread 用来负责管理 DispSyncSource 延时源，由于分别创建了用于 APP 绘制和 SF 合成的 DispSyncSource 源，因此也对应创建了两个分别用于管理它们的 EventThread 线程。

```
// class EventThread : public android::EventThread, private VSyncSource::Callback {
//    class Connection : public BnDisplayEventConnection {}
// }
// class BnDisplayEventConnection : public SafeBnInterface<IDisplayEventConnection>

EventThread::EventThread(VSyncSource* src, ResyncWithRateLimitCallback resyncWithRateLimitCallback,
        InterceptVSyncsCallback interceptVSyncsCallback, const char* threadName) : mVSyncSource(src),
        mResyncWithRateLimitCallback(resyncWithRateLimitCallback), mInterceptVSyncsCallback(interceptVSyncsCallback) {
    mThread = std::thread(&EventThread::threadMain, this);
    // ...
}

void EventThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    while (mKeepRunning) {
        // 等待唤醒
        signalConnections = waitForEventLocked(&lock, &event);
        // ...
    }
}
```

### ET.waitForEventLocked

启动了 EventThread 后，调用 threadMain 方法，我们看一下 waitForEventLocked 方法的注释: `This will return when (1) a vsync event has been received, and (2) there was at least one connection interested in receiving it when we started waiting.` 即 waitForEventLocked 方法只有当**接收到了 Vsync 信号**且**至少有一个 Connection 正在等待 Vsync 信号**才会返回，否则会调用 mCondition.wait/wait\_for 方法一直等待：

```
Vector<sp<EventThread::Connection> > EventThread::waitForEventLocked(
        std::unique_lock<std::mutex>* lock, DisplayEventReceiver::Event* event) {
    Vector<sp<EventThread::Connection> > signalConnections;
    while (signalConnections.isEmpty() && mKeepRunning) {
        bool eventPending = false;
        bool waitForVSync = false;

        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        for (int32_t i = 0; i < DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES; i++) {
            timestamp = mVSyncEvent[i].header.timestamp;
            if (timestamp) {
                // we have a vsync event to dispatch
                if (mInterceptVSyncsCallback) {
                    mInterceptVSyncsCallback(timestamp);
                }
                *event = mVSyncEvent[i];
                mVSyncEvent[i].header.timestamp = 0;
                vsyncCount = mVSyncEvent[i].vsync.count;
                break;
            }
        }

        // 查找正在等待连接的event
        size_t count = mDisplayEventConnections.size();
        for (size_t i = 0; i < count;) {
            sp<Connection> connection(mDisplayEventConnections[i].promote());
            if (connection != nullptr) {
                bool added = false;
                if (connection->count >= 0) {
                    // 需要 vsync 事件，因为至少有一个连接正在等待vsync
                    waitForVSync = true;
                    if (timestamp) {
                        // we consume the event only if it's time (ie: we received a vsync event)
                        if (connection->count == 0) {
                            // fired this time around
                            connection->count = -1;
                            signalConnections.add(connection);
                            added = true;
                        } else if (connection->count == 1 || (vsyncCount % connection->count) == 0) {
                            // continuous event, and time to report it
                            signalConnections.add(connection);
                            added = true;
                        }
                    }
                }
                if (eventPending && !timestamp && !added) {
                    // 没有vsync事件需要处理但存在pending消息
                    signalConnections.add(connection);
                }
                ++i;
            } else {
                // 该连接已死亡则直接移除
                mDisplayEventConnections.removeAt(i);
                --count;
            }
        }

        if (timestamp && !waitForVSync) {
            // 接收到 Vsync 事件，但是没有 client 需要它
            disableVSyncLocked();
        } else if (!timestamp && waitForVSync) {
            // Vsync 事件还没到来且至少存在一个 client
            enableVSyncLocked();
        }

        if (!timestamp && !eventPending) {
            if (waitForVSync) {
                // 等待vsync事件和新的client注册，当vsync发生后，会调用mCondition.notify
                // If the screen is off, we can't use h/w vsync, so we
                // use a 16ms timeout instead.  It doesn't need to be
                // precise, we just need to keep feeding our clients.
                bool softwareSync = mUseSoftwareVSync;
                auto timeout = softwareSync ? 16ms : 1000ms;
                if (mCondition.wait_for(*lock, timeout) == std::cv_status::timeout) {
                    // ...
                }
            } else {
                // 线程等待
                mCondition.wait(*lock);
            }
        }
    }
    // 执行到这里，表示存在需要 vsync 的连接以及收到了 vsync 事件
    return signalConnections;
}
复制代码
```

看一下 disableVSyncLocked 和 enableVSyncLocked 的实现：

```
void EventThread::enableVSyncLocked() {
    if (!mUseSoftwareVSync) {
        // never enable h/w VSYNC when screen is off
        if (!mVsyncEnabled) {
            mVsyncEnabled = true;
            mVSyncSource->setCallback(this);
            mVSyncSource->setVSyncEnabled(true);
        }
    }
}

void EventThread::disableVSyncLocked() {
    if (mVsyncEnabled) {
        mVsyncEnabled = false;
        mVSyncSource->setVSyncEnabled(false);
    }
}
复制代码
```

可以看出 enableVSyncLocked 和 disableVSyncLocked 这两个方法的作用：**通过 DispSyncSource 同步源向 DispSync 同步模型的 DispSyncThread 线程添加或移除对 DispSync 同步模型中 Vsync 信号的监听器。同时 enableVSyncLocked 方法还通过 DispSyncSource.setCallback 方法设置了在 DispSyncSource 延时源收到来自 DispSync 同步模型的 Vsync 信号后的监听者**。

Vsync 事件到达后会将其保存在参数 mVSyncEvent 中，如果 timestamp 为 0 则表示没有 Vsync 信号到达。在 mDisplayEventConnections 中保存了注册的监听者，如果 `connection->count >= 0` 则表示有监听者对 Vsync 信号感兴趣，于是 将 waitForVSync 置为 true 且将该监听者添加到 signalConnections 集合中。根据源码注释可知 `connection.count` 的含义如下：

* count >= 1: continuous event. count is the vsync rate -- 会持续接收 Vsync 事件
* count == 0: one-shot event that has not fired -- 只接受一次 Vsync 事件，初始化时 count 为 0
* count == -1: one-shot event that fired this round / disabled -- 不再接收 Vsync 事件

在取得了 timestamp 和 waitForVSync 的值后：

* **若 Vsync 信号已经到达但没有感兴趣的监听者** 则 **通过 disableVSyncLocked 方法移除 DispSyncSource 对 DispSync 同步模型的 Vsync 信号的监听**
* **若有感兴趣的监听者但 Vsync 信号还未达到** 则 **通过 enableVSyncLocked 方法将 DispSyncSource 添加到 DispSync 同步模型的监听者集合**
* **若 Vsync 信号未到达且没有其他等待的事件** 则 **如果 waitForVSync == true 则要等待 Vsync 信号(不会无限等待，有超时时间)，否则阻塞当前线程**

### MQ.setEventThread

接下来 SF 初始化过程中调用了 MessageQueue.setEventThread 方法：

```
// sp<IDisplayEventConnection> mEvents;
void MessageQueue::setEventThread(android::EventThread* eventThread) {
    if (mEventThread == eventThread) {
        return;
    }

    if (mEventTube.getFd() >= 0) {
        mLooper->removeFd(mEventTube.getFd());
    }

    mEventThread = eventThread;
    mEvents = eventThread->createEventConnection(); // 创建连接
    mEvents->stealReceiveChannel(&mEventTube);
    // mEventTube 是 BitTube 对象
    // 监听 mEventTube(BitTube), 一旦有数据到来则调用 cb_eventReceiver 方法
    mLooper->addFd(mEventTube.getFd(), 0, Looper::EVENT_INPUT, MessageQueue::cb_eventReceiver, this);
}

sp<BnDisplayEventConnection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}

// gui::BitTube mChannel;
status_t EventThread::Connection::stealReceiveChannel(gui::BitTube* outChannel) {
    outChannel->setReceiveFd(mChannel.moveReceiveFd());
    return NO_ERROR;
}

// 创建 Connection 后调用
void EventThread::Connection::onFirstRef() {
    mEventThread->registerDisplayEventConnection(this);
}

status_t EventThread::registerDisplayEventConnection(
        const sp<EventThread::Connection>& connection) {
    std::lock_guard<std::mutex> lock(mMutex);
    mDisplayEventConnections.add(connection); // 将 connection 加入连接列表
    mCondition.notify_all();
    return NO_ERROR;
}
```

**可以看到 MQ.setEventThread 方法创建了一个 Connection 连接且将其加入了 mDisplayEventConnections 列表**。接着调用 Looper.addFd 方法监听 mEventTube 中的数据，下面看下这个方法：

```
int Looper::addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data) {
    return addFd(fd, ident, events, callback ? new SimpleLooperCallback(callback) : NULL, data);
}

int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
    if (!callback.get()) {
        if (!mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }
        if (ident < 0) {
            ALOGE("Invalid attempt to set NULL callback with ident < 0.");
            return -1;
        }
    } else {
        ident = POLL_CALLBACK; // -2
    }
    Request request;
    request.fd = fd;
    request.ident = ident;
    request.events = events;
    request.callback = callback;
    request.data = data;
    ssize_t requestIndex = mRequests.indexOfKey(fd);
    if (requestIndex < 0) { // 之前不存在这个request
        mRequests.add(fd, request);
    } else { // 之前存在则替换
        mRequests.replaceValueAt(requestIndex, request);
    }
}

SimpleLooperCallback::SimpleLooperCallback(Looper_callbackFunc callback) : mCallback(callback) {
}

int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    return mCallback(fd, events, data);
}
```

调用 Looper.addFd 方法会根据传入的参数创建一个 request 对象，并将其加入 mRequests 列表。接下来会在 poll 方法中取出：

```
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, NULL, NULL, NULL);
}

int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) { // 死循环
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) { // ident >= 0 则返回
                // ...
                return ident;
            }
        }
        if (result != 0) {
            return result;
        }
        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
    // ...
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            // Invoke the callback.
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                // handleEvent 返回 0 则会移除该 fd, 而在 Choreographer 逻辑中会返回 1 以一直保持 callback
                removeFd(fd, response.request.seq);
            }
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

因此，当调用 addFd 函数时传入的 fd 有数据接收时，则会根据传入的 callback 参数执行不同的逻辑：

* callback 是一个函数时则 `response.request.callback->handleEvent` 直接调用该函数
* callback 是一个对象时则 `response.request.callback->handleEvent` 调用的是该对象自定义的 handleEvent 方法

### SurfaceFlinger.run

在 SurfaceFlinger 初始化完成后会调用其 run 方法：

```
void SurfaceFlinger::run() {
    do { // 循环等待事件
        waitForEvent();
    } while (true);
}

void SurfaceFlinger::waitForEvent() {
    mEventQueue->waitMessage();
}

void MessageQueue::waitMessage() {
    do {
        IPCThreadState::self()->flushCommands();
        int32_t ret = mLooper->pollOnce(-1);
        // ...
    } while (true);
}
```

可以看到 SurfaceFlinger 主线程通过死循环执行 MessageQueue.waitMessage 方法等待消息的到来，其内部调用的便是上面看过的 Looper.pollOnce 方法。

### ET.onVSyncEvent

上面当 DispSync 同步模型产生 Vsync 信号后，会通知 DispSyncSource 源，进而回调监听者(EventThread)的 onVSyncEvent 方法。

```
void EventThread::onVSyncEvent(nsecs_t timestamp) {
    std::lock_guard<std::mutex> lock(mMutex);
    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
    mVSyncEvent[0].header.id = 0;
    mVSyncEvent[0].header.timestamp = timestamp;
    mVSyncEvent[0].vsync.count++;
    mCondition.notify_all();
}
```

onVSyncEvent 方法中执行 mCondition.notify\_all 唤醒了 EventThread 线程，接着上面 EventThread 线程的逻辑开始看：

```
void EventThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    std::unique_lock<std::mutex> lock(mMutex);
    while (mKeepRunning) {
        DisplayEventReceiver::Event event;
        Vector<sp<EventThread::Connection> > signalConnections;
        signalConnections = waitForEventLocked(&lock, &event);

        // dispatch events to listeners...
        const size_t count = signalConnections.size();
        for (size_t i = 0; i < count; i++) {
            const sp<Connection>& conn(signalConnections[i]);
            // now see if we still need to report this event
            status_t err = conn->postEvent(event);
            // ...
        }
    }
}

status_t EventThread::Connection::postEvent(const DisplayEventReceiver::Event& event) {
    ssize_t size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}

ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel, Event const* events, size_t count) {
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```

这里通过 BitTube::sendObjects 发送数据，根据前面 addFd 方法的解析，当接收到数据时，会调用 MessageQueue::cb\_eventReceiver 方法：

```
int MessageQueue::cb_eventReceiver(int fd, int events, void* data) {
    MessageQueue* queue = reinterpret_cast<MessageQueue*>(data);
    return queue->eventReceiver(fd, events);
}

int MessageQueue::eventReceiver(int /*fd*/, int /*events*/) {
    ssize_t n;
    DisplayEventReceiver::Event buffer[8];
    while ((n = DisplayEventReceiver::getEvents(&mEventTube, buffer, 8)) > 0) {
        for (int i = 0; i < n; i++) {
            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
                mHandler->dispatchInvalidate();
                break;
            }
        }
    }
    return 1;
}

void MessageQueue::Handler::dispatchInvalidate() {
    if ((android_atomic_or(eventMaskInvalidate, &mEventMask) & eventMaskInvalidate) == 0) {
        mQueue.mLooper->sendMessage(this, Message(MessageQueue::INVALIDATE));
    }
}

void MessageQueue::Handler::handleMessage(const Message& message) {
    switch (message.what) {
        case INVALIDATE:
            android_atomic_and(~eventMaskInvalidate, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
        case REFRESH:
            android_atomic_and(~eventMaskRefresh, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
    }
}
```

于是进入 SF.onMessageReceived 方法，开始进行图形合成输出逻辑。

### 图像合成流程

### SF.onMessageReceived

```
void SurfaceFlinger::onMessageReceived(int32_t what) {
    switch (what) {
        case MessageQueue::INVALIDATE: {
            bool frameMissed = !mHadClientComposition && mPreviousPresentFence != Fence::NO_FENCE &&
                    (mPreviousPresentFence->getSignalTime() == Fence::SIGNAL_TIME_PENDING);
            if (frameMissed) {
                mTimeStats.incrementMissedFrames();
                if (mPropagateBackpressure) {// 丢帧且Backpressure则跳过此次Transaction和refresh
                    signalLayerUpdate();
                    break;
                }
            }
            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate();
            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded) {
                // Signal a refresh if a transaction modified the window state, a new buffer was latched, 
                // or if HWC has requested a full repaint
                // 最终会调用 SF.handleMessageRefresh 方法
                signalRefresh();
            }
            break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
    }
}
```

### SF.signalLayerUpdate

```
void SurfaceFlinger::signalLayerUpdate() {
    mEventQueue->invalidate();
}

void MessageQueue::invalidate() {
    // mEvents 是 EventThread.Connection 类型
    mEvents->requestNextVsync(); // 请求下一次Vsync信号
}

void EventThread::Connection::requestNextVsync() {
    mEventThread->requestNextVsync(this);
}

void EventThread::requestNextVsync(const sp<EventThread::Connection>& connection) {
    std::lock_guard<std::mutex> lock(mMutex);

    if (mResyncWithRateLimitCallback) {
        mResyncWithRateLimitCallback();
    }

    if (connection->count < 0) {
        connection->count = 0;
        // 唤醒EventThread线程
        mCondition.notify_all();
    }
}
```

这里看一下 mResyncWithRateLimitCallback 参数，它是在 SF 初始化创建 EventThread 时传入的，`mResyncWithRateLimitCallback()` 调用的是 SF.resyncWithRateLimit 方法：

```
// SF 向硬件请求 Vsync 的间隔必须大于 500ns, 否则忽略
void SurfaceFlinger::resyncWithRateLimit() {
    static constexpr nsecs_t kIgnoreDelay = ms2ns(500);

    static nsecs_t sLastResyncAttempted = 0;
    const nsecs_t now = systemTime();
    if (now - sLastResyncAttempted > kIgnoreDelay) {
        resyncToHardwareVsync(false);
    }
    sLastResyncAttempted = now;
}
```

因此 signalLayerUpdate 方法的作用是请求接收下一次 Vsync 信号。

### SF.handleMessageTransaction

```
bool SurfaceFlinger::handleMessageTransaction() {
    uint32_t transactionFlags = peekTransactionFlags();
    if (transactionFlags) {
        handleTransaction(transactionFlags);
        return true;
    }
    return false;
}

void SurfaceFlinger::handleTransaction(uint32_t transactionFlags)
{
    // ...
    transactionFlags = getTransactionFlags(eTransactionMask);
    // 调用每个 Layer 的 doTransaction 方法，处理 layers 的改变
    handleTransactionLocked(transactionFlags);
}
```

### SF.handleMessageInvalidate

```
bool SurfaceFlinger::handleMessageInvalidate() {
    ATRACE_CALL();
    // Store the set of layers that need updates -- mLayersWithQueuedFrames.
    // 存储需要更新的 layers
    return handlePageFlip();
}
```

### SF.handleMessageRefresh

```
void SurfaceFlinger::handleMessageRefresh() {
    nsecs_t refreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);
    // 如果图层有更新则执行 invalidate 过程
    preComposition(refreshStartTime);
    // 重建每个显示屏的所有可见的 Layer 列表
    rebuildLayerStacks();
    // 更新 HWComposer 的 Layer
    setUpHWComposer();
    // 合成所有 Layer 的图像
    doComposition();
    // 回调每个 layer 的 onPostComposition 方法
    postComposition(refreshStartTime);
    // 清空需要更新的 layers 列表
    mLayersWithQueuedFrames.clear();
}
```

代码过多，篇幅所限，具体的代码不贴出来了，需要的话可以去 SurfaceFlinger.cpp 的源码中查阅。

### 总结

在 SurfaceFlinger 的启动流程中：

1. 首先会创建 SurfaceFlinger 对象，在构造器中创建了 DispSync 同步模型对象；
2. 然后执行初始化 SurfaceFlinger 的逻辑：
   * 注册监听，接收 HWC 的相关事件。
   * 启动 APP 和 SF 的 EventThread 线程，用来管理基于 DispSync 创建的两个 DispSyncSource 延时源对象，分别是用于绘制(app--mEventThreadSource)和合成(SurfaceFlinger--mSfEventThreadSource)。启动了 EventThread 线程后，会一直阻塞在 waitForEventLocked 方法中(期间会根据需要设置监听器)，直到**接收到 Vsync 信号**且**至少有一个连接正在等待 Vsync 信号**才会继续执行线程逻辑，即通知监听者；
   * 通过 MessageQueue.setEventThread 方法创建了一个连接，并通过 Looper.addFd 方法监听 BitTube 数据。
   * 创建 HWComposer 对象(通过 HAL 层的 HWComposer 硬件模块 或 软件模拟产生 Vsync 信号)，**现在的 Android 系统基本上都可以看成是通过硬件 HWComposer 产生 Vsync 信号，而不使用软件模拟，所以下面解析都只谈及硬件 HWComposer 的 Vsync 信号**；
   * 初始化非虚拟的显示屏；
   * 启动开机动画服务；
3. 最后执行 SurfaceFlinger.run 逻辑，该方法会在 SurfaceFlinger 主线程通过死循环执行 MessageQueue.waitMessage 方法等待消息的到来，其内部调用了 Looper.pollOnce 方法，该方法会从 Looper.addFd 方法监听的 BitTube 中读取数据，当有数据到来时执行对应的回调方法。

当硬件或软件模拟发出 Vsync 信号时：

1. 回调 SF 相关方法，SF 调用 DispSync 同步模型的方法处理 Vsync 信号(统计和计算模型的偏移和周期)，并根据返回值判断是否使能/关闭 HWC Vsync 信号的发出。
2. DispSync 根据计算的偏移和周期计算下次 Vsync 信号发生时间，并通知监听者 Vsync 信号到达的事件，传递给 DispSyncSource 延时源，延时源通过 EventThread 来管理 Vsync 信号的收发。
3. EventThread 调用连接 Connection 对象向 BitTube 发送数据，触发 addFd 函数中设置的回调方法，回调方法进而调用 SF.onMessageReceived 函数，然后进行图像的合成等工作。

另一方面，Choreographer 会通过上面创建的 APP 延时源 mEventThreadSource 对象及其对应的 EventThread 线程来监听同步模拟发出的 Vsync 信号，然后进行绘制(measure/layout/draw)操作。

将 SurfaceFlinger 的工作流程总结如下图：

![SurfaceFlinger工作流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5aba0725e87d42f08d8af7a91eed6870~tplv-k3u1fbpfcp-zoom-1.image)



[https://wizzie.top/Blog/2019/09/21/2019/190921-android-wms-view-cpp/](https://wizzie.top/Blog/2019/09/21/2019/190921-android-wms-view-cpp/)
