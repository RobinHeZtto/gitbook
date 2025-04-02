# ANR

### 1. 概念

ANR(Application Not Responding) 应用程序无响应。当应用程序长时间被阻塞无法响应用户操作时，就会出现ANR，通常系统会弹出一个提示提示框，提示用户程序无响应，是否等待或关闭。

### 2. ANR类型及产生原因

#### 2.1 类型：

* **InputDispatcher Timeout（常见）：**&#x69;nput事件&#x5728;**`5S`**&#x5185;没有处理完成导致ANR（Input的超时机制与其他的不同，对于input来说即便某次事件执行时间超过timeout时长，只要用户后续在没有再生成输入事件，则不会触发ANR）。 logcat日志关键字：`Input dispatching timed out`
* **BroadcastTimeout ：**&#x524D;台Broadcast：onReceiver&#x5728;**`10S`**&#x5185;没有处理完成发生ANR。 后台Broadcast：onReceiver&#x5728;**`60`**`s`内没有处理完成发生ANR。 logcat日志关键字：`Timeout of broadcast`&#x20;
* **ServiceTimeout：**&#x524D;台Service：`onCreate`，`onStart`，`onBind`等生命周期&#x5728;**`20s`**&#x5185;没有处理完成发生ANR。 后台Service：`onCreate`，`onStart`，`onBind`等生命周期&#x5728;**`200s`**&#x5185;没有处理完成发生ANR 。logcat日志关键字：`Timeout executing service`
* **ContentProviderTimeout ：**&#x43;ontentProvider publish&#x5728;**`10S`**&#x5185;没有处理完成发生ANR，则会直接杀进程以及清理相应信息，而不会弹出ANR的对话框。 logcat日志关键字：`timeout publishing content providers`

|            类型           |  前台 |  后台  |
| :---------------------: | :-: | :--: |
| InputDispatcher Timeout |  5s |  NA  |
|    Broadcast Timeout    | 10s |  60s |
|     Service Timeout     | 20s | 200s |
|  ConentProvider Timeout | 10s |  10s |

#### 2.2 产生原因：

可能造成出现ANR的原因：

* 主线程进行耗时操作，如网络/IO/序列化等；
* 多线程操作造成死锁，主线程被block；
* 主线程被Binder对端block；
* service binder的连接达到上线无法和和System Server通信；
* 系统资源紧张导致阻塞（CPU、IO、管道）
* System Server中WatchDog出现ANR

从进程的角度看，原因有：

* 当前进程：主线程本身耗时或者主线程的消息队列存在耗时操作、主线程被本进程的其它子线程所blocked
* 远端进程：binder call、socket通信

### 3. ANR触发机制

从系统级去梳理清晰ANR的来龙去脉，对ANR有一个全面、正确的理解，对处理ANR问题大有益处。

> 该部分详细可参考gityuan博客：[彻底理解安卓应用无响应机制](https://zhuanlan.zhihu.com/p/62509258)

ANR是一套监控Android应用响应是否及时的机制，可以把发生ANR比作是`埋炸弹`、`拆炸弹`、`引爆炸弹`，整个流程：

1. **埋定时炸弹：**&#x4EFB;务开始时，中控系统(system\_server进程)启动倒计时，在规定时间内如果目标(应用进程)没有干完所有的活，则中控系统会定向引爆目标(杀死应用进程)。
2. **拆炸弹：**&#x5728;规定的时间内如果目标（应用进程）干完工地的所有活，并及时向中控系统报告完成，请求解除定时炸弹，则幸免于难。
3. **引爆炸弹：**&#x5982;果在规定的时间内应用进程没有干完工地的所有活，中控系统立即封装现场，抓取快照，搜集目标执行慢的罪证(traces)，便于后续的案件侦破(调试分析)，最后是炸毁目标。

#### 3.1 ServiceTimeout:

埋炸弹与拆炸弹在整个服务启动(startService)过程所处的环节:

![](<../../.gitbook/assets/image (473).png>)

图解：

1. 客户端(App进程)向中控系统(system\_server进程)发起启动服务的请求
2. 中控系统派出一名空闲的通信员(binder\_1线程)接收该请求，紧接着向组件管家(ActivityManager线程)发送消息，埋下定时炸弹
3. 通讯员1号(binder\_1)通知工地(service所在进程)的通信员准备开始干活
4. 通讯员3号(binder\_3)收到任务后转交给包工头(main主线程)，加入包工头的任务队列(MessageQueue)
5. 包工头经过一番努力干完活(完成service启动的生命周期)，然后等待SharedPreferences(简称SP)的持久化；
6. 包工头在SP执行完成后，立刻向中控系统汇报工作已完成
7. 中控系统的通讯员2号(binder\_2)收到包工头的完工汇报后，立刻拆除炸弹。如果在炸弹倒计时结束前拆除炸弹则相安无事，否则会引发爆炸(触发ANR)

#### 3.2 BroadcastTimeout:

broadcast跟service超时机制大抵相同，对于静态注册的广播在超时检测过程需要检测SP，如下图所示。

![](<../../.gitbook/assets/image (400).png>)

图解：

1. 客户端(App进程)向中控系统(system\_server进程)发起发送广播的请求
2. 中控系统派出一名空闲的通信员(binder\_1)接收该请求转交给组件管家(ActivityManager线程)
3. 组件管家执行任务(processNextBroadcast方法)的过程埋下定时炸弹
4. 组件管家通知工地(receiver所在进程)的通信员准备开始干活
5. 通讯员3号(binder\_3)收到任务后转交给包工头(main主线程)，加入包工头的任务队列(MessageQueue)
6. 包工头经过一番努力干完活(完成receiver启动的生命周期)，发现当前进程还有SP正在执行写入文件的操作，便将向中控系统汇报的任务交给SP工人(queued-work-looper线程)
7. SP工人历经艰辛终于完成SP数据的持久化工作，便可以向中控系统汇报工作完成
8. 中控系统的通讯员2号(binder\_2)收到包工头的完工汇报后，立刻拆除炸弹。如果在倒计时结束前拆除炸弹则相安无事，否则会引发爆炸(触发ANR)

（说明：SP从8.0开始采用名叫“queued-work-looper”的handler线程，在老版本采用newSingleThreadExecutor创建的单线程的线程池）

如果是动态广播，或者静态广播没有正在执行持久化操作的SP任务，则不需要经过“queued-work-looper”线程中转，而是直接向中控系统汇报，流程更为简单，如下图所示：

![](<../../.gitbook/assets/image (482).png>)

可见，只有XML静态注册的广播超时检测过程会考虑是否有SP尚未完成，动态广播并不受其影响。SP的apply将修改的数据项更新到内存，然后再异步同步数据到磁盘文件，因此很多地方会推荐在主线程调用采用apply方式，避免阻塞主线程，但静态广播超时检测过程需要SP全部持久化到磁盘，如果过度使用apply会增大应用ANR的概率，更多细节详见[http://gityuan.com/2017/06/18/SharedPreferences](https://link.zhihu.com/?target=http%3A//gityuan.com/2017/06/18/SharedPreferences)

Google这样设计的初衷是针对静态广播的场景下，保障进程被杀之前一定能完成SP的数据持久化。因为在向中控系统汇报广播接收者工作执行完成前，该进程的优先级为Foreground级别，高优先级下进程不但不会被杀，而且能分配到更多的CPU时间片，加速完成SP持久化。

#### 3.3 ContentProviderTimeout :

ContentProviderTimeout是在provider进程首次启动的时候才会检测，当provider进程已启动的场景，再次请求provider并不会触发provider超时。

![](<../../.gitbook/assets/image (486).png>)

图解：

1. 客户端(App进程)向中控系统(system\_server进程)发起获取内容提供者的请求
2. 中控系统派出一名空闲的通信员(binder\_1)接收该请求，检测到内容提供者尚未启动，则先通过zygote孵化新进程
3. 新孵化的provider进程向中控系统注册自己的存在
4. 中控系统的通信员2号接收到该信息后，向组件管家(ActivityManager线程)发送消息，埋下炸弹
5. 通信员2号通知工地(provider进程)的通信员准备开始干活
6. 通讯员4号(binder\_4)收到任务后转交给包工头(main主线程)，加入包工头的任务队列(MessageQueue)
7. 包工头经过一番努力干完活(完成provider的安装工作)后向中控系统汇报工作已完成
8. 中控系统的通讯员3号(binder\_3)收到包工头的完工汇报后，立刻拆除炸弹。如果在倒计时结束前拆除炸弹则相安无事，否则会引发爆炸(触发ANR)

#### 3.4 InputDispatcher **Timeout:**

input的超时检测机制跟service、broadcast、provider截然不同，为了更好的理解input过程先来介绍两个重要线程的相关工作：

* **InputReader**线程负责通过EventHub(监听目录/dev/input)读取输入事件，一旦监听到输入事件则放入到**InputDispatcher**的`mInBoundQueue`队列，并通知其处理该事件；
* InputDispatcher线程负责将接收到的输入事件分发给目标应用窗口，分发过程使用到3个事件队列：
  * `mInBoundQueue`用于记录InputReader发送过来的输入事件；
  * `outBoundQueue`用于记录即将分发给目标应用窗口的输入事件；
  * `waitQueue`用于记录已分发给目标应用，且应用尚未处理完成的输入事件；

input的超时机制并非时间到了一定就会爆炸，而是处理后续上报事件的过程才会去检测是否该爆炸，所以更相信是扫雷的过程，具体如下图所示。

![](<../../.gitbook/assets/image (167).png>)

1. InputReader线程通过EventHub监听底层上报的输入事件，一旦收到输入事件则将其放至`mInBoundQueue`队列，并唤醒InputDispatcher线程
2. InputDispatcher开始分发输入事件，设置埋雷的起点时间。先检测是否有正在处理的事件(mPendingEvent)，如果没有则取出`mInBoundQueue`队头的事件，并将其赋值给mPendingEvent，且重置ANR的timeout；否则不会从mInBoundQueue中取出事件，也不会重置timeout。然后检查窗口是否就绪(checkWindowReadyForMoreInputLocked)，满足以下任一情况，则会进入扫雷状态(检测前一个正在处理的事件是否超时)，终止本轮事件分发，否则继续执行步骤3。

* 对于按键类型的输入事件，则outboundQueue或者waitQueue不为空，
* 对于非按键的输入事件，则waitQueue不为空，且等待队头时间超时500ms

1. 当应用窗口准备就绪，则将mPendingEvent转移到`outBoundQueue`队列
2. 当`outBoundQueue`不为空，且应用管道对端连接状态正常，则将数据从`outboundQueue`中取出事件，放入waitQueue队列
3. InputDispatcher通过socket告知目标应用所在进程可以准备开始干活
4. App在初始化时默认已创建跟中控系统双向通信的socketpair，此时App的包工头(main线程)收到输入事件后，会层层转发到目标窗口来处理
5. 包工头完成工作后，会通过socket向中控系统汇报工作完成，则中控系统会将该事件从`waitQueue`队列中移除。

input超时机制为什么是扫雷，而非定时爆炸呢？是由于对于input来说即便某次事件执行时间超过timeout时长，只要用户后续在没有再生成输入事件，则不会触发ANR。 这里的扫雷是指当前输入系统中正在处理着某个耗时事件的前提下，后续的每一次input事件都会检测前一个正在处理的事件是否超时（进入扫雷状态），检测当前的时间距离上次输入事件分发时间点是否超过timeout时长。如果前一个输入事件，则会重置ANR的timeout，从而不会爆炸。

#### 3.5 ANR爆炸过程：

对于service、broadcast、provider、input发生ANR后，中控系统会马上去抓取现场的信息，用于调试分析。收集的信息包括如下：

* 将`am_anr`信息输出到EventLog，也就是说**ANR触发的时间点最接近的就是EventLog中输出的am\_anr信息**。
* 收集以下重要进程的各个线程调用栈trace信息，保存在data/anr/traces.txt文件
  * 当前发生ANR的进程，system\_server进程以及所有persistent进程。
  * audioserver, cameraserver, mediaserver, surfaceflinger等重要的native进程。
  * CPU使用率排名前5的进程。\

* 将发生ANR的reason以及CPU使用情况信息输出到main log。
* 将traces文件和CPU使用情况信息保存到dropbox，即`data/system/dropbox`目录。
* 对用户可感知的进程则弹出ANR对话框告知用户，对用户不可感知的进程发生ANR则直接杀掉。

### 4. 定位与分析

#### 4.1 分析方法：

1.确认cpu loading， iowait，锁等资源问题引起的ANR。

分析`trace.txt`，查看是否可能是因为**CPU loading**，**IO wait**，**耗时堆栈调用**，**主线程等待持锁**导致ANR。trace关键信息：

* 各个应用进程CPU Usage占比: `CPU usage`
* IO wait：`iowait`&#x20;
* 耗时堆栈调用：main线程的堆栈信息，根据具体代码分析
* 锁：`waiting on` `waiting to lock`

分析`anroid.log`，搜索关键字`ANR in`查看CPU loading信息，如下

```
Load: 3.0 / 4.0 / 3.0 (Load表明是1分钟,5分钟,15分钟CPU的负载)
```

2\. 如不能通过trace直接定位到是否是系统资源引起，查看**EventLog，**&#x786E;认**ANR时间、进程PID、ANR类型**（EventLog中输出的am\_anr时间点最接近ANR触发的时间点）。

```bash
$ grep am_anr event.log
12-17 10:11:01.521  1681 31735 I am_anr  : [0,26662,net.oneplus.weather,814235206,
Broadcast of Intent { act=net.oneplus.weather.receiver.BootReceiver.ACTION_ALARM 
flg=0x14 pkg=net.oneplus.weather cmp=net.oneplus.weather/.receiver.AlarmReceiver
 (has extras) }]
```

从上面的Event log可以看到：

* **ANR时间：**&#x31;2-16 08:51:02.857
* **进程PID：**&#x32;6662
* **ANR类型：**&#x42;roadcast Timeout

根据获取到的ANR的进程PID，从`android.log`中过滤出所有该进程的log信息，然后根据ANR类型及ANR时间确定造成ANR的起始时间，然后从该时间对照代码排查log。例如上述例子为Broadcast Timeout，那么则应排查`12-17 10:11:00.521` 到 `12-17 10:11:01.521`时间内`26662`进程的log。

3\. 如果无法根据应用业务log确认问题，继续排查是否是因为binder通信失败导致。排查关键字`binder thread pool`或`transaction failed`

binder线程池占满而无法与system\_server通信的情况：

```
64247:12-17 09:56:20.888 26662 27432 E IPCThreadState: binder thread pool (15 threads) starved for 601 ms
```

binder通信失败的情况：

```
07-21 06:04:35.580 [32837.690321] binder: 1698:2362 transaction failed 29189/-3, size 100-0 line 3042
```

#### 4.2 trace.txt文件解读

```
"main" prio=5 tid=1 Runnable
  | group="main" sCount=0 dsCount=0 obj=0x73bcc7d0 self=0x7f20814c00
  | sysTid=20176 nice=-10 cgrp=default sched=0/0 handle=0x7f251349b0
  | state=R schedstat=( 0 0 0 ) utm=12 stm=3 core=5 HZ=100
  | stack=0x7fdb75e000-0x7fdb760000 stackSize=8MB
  | held mutexes= "mutator lock"(shared held)
  // java 堆栈调用信息,可以查看调用的关系，定位到具体位置
  at ttt.push.InterceptorProxy.addMiuiApplication(InterceptorProxy.java:77)
  at ttt.push.InterceptorProxy.create(InterceptorProxy.java:59)
  at android.app.Activity.onCreate(Activity.java:1041)
  at miui.app.Activity.onCreate(SourceFile:47)
  at com.xxxx.moblie.ui.b.onCreate(SourceFile:172)
  at com.xxxx.moblie.ui.MainActivity.onCreate(SourceFile:68)
  at android.app.Activity.performCreate(Activity.java:7050)
  at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1214)
  at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2807)
  at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2929)
  at android.app.ActivityThread.-wrap11(ActivityThread.java:-1)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1618)
  at android.os.Handler.dispatchMessage(Handler.java:105)
  at android.os.Looper.loop(Looper.java:171)
  at android.app.ActivityThread.main(ActivityThread.java:6699)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:246)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:783)
```

```
"Binder_1" prio=5 tid=8 Native
  | group="main" sCount=1 dsCount=0 obj=0x12c610a0 self=0x5573e5c750
  | sysTid=12092 nice=0 cgrp=default sched=0/0 handle=0x7fa2743450
  | state=S schedstat=( 796240075 863170759 3586 ) utm=50 stm=29 core=1 HZ=100
  | stack=0x7fa2647000-0x7fa2649000 stackSize=1013KB
  | held mutexes=
```

* 第0行:
  * 线程名: Binder\_1（如有daemon则代表守护线程)
  * prio: 线程优先级
  * tid: 线程内部id
  * 线程状态: NATIVE
* 第1行:
  * group: 线程所属的线程组
  * sCount: 线程挂起次数
  * dsCount: 用于调试的线程挂起次数
  * obj: 当前线程关联的java线程对象
  * self: 当前线程地址
* 第2行：
  * sysTid：线程真正意义上的tid
  * nice: 调度有优先级
  * cgrp: 进程所属的进程调度组
  * sched: 调度策略
  * handle: 函数处理地址
* 第3行：
  * state: 线程状态
  * schedstat: CPU调度时间统计
  * utm/stm: 用户态/内核态的CPU时间(单位是jiffies)
  * core: 该线程的最后运行所在核
  * HZ: 时钟频率
* 第4行：
  * stack：线程栈的地址区间
  * stackSize：栈的大小
* 第5行：
  * mutex: 所持有mutex类型，有独占锁exclusive和共享锁shared两类
* schedstat含义说明：
  * nice值越小则优先级越高。此处nice=-2, 可见优先级还是比较高的;
  * schedstat括号中的3个数字依次是Running、Runable、Switch，紧接着的是utm和stm
    * Running时间：CPU运行的时间，单位ns
    * Runable时间：RQ队列的等待时间，单位ns
    * Switch次数：CPU调度切换次数
    * utm: 该线程在用户态所执行的时间，单位是jiffies，jiffies定义为sysconf(\_SC\_CLK\_TCK)，默认等于10ms
    * stm: 该线程在内核态所执行的时间，单位是jiffies，默认等于10ms
* 可见，该线程Running=186667489018ns,也约等于186667ms。在CPU运行时间包括用户态(utm)和内核态(stm)。 utm + stm = (12112 + 6554) ×10 ms = 186666ms。
* 结论：utm + stm = schedstat第一个参数值。

### 5. ANR监测

1.使用[FileObserver](http://tandroidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/os/FileObserver.java)监听 /data/anr/traces.txt的变化

* 缺点：高版本ROM需要root权限
* 解决方案：海外Google Play服务、国内[Hardcoder](https://github.com/Tencent/Hardcoder)

2.监控消息队列的运行时间

利用主线程的消息队列处理机制，应用发生卡顿，一定是在dispatchMessage中执行了耗时操作。我们通过给主线程的Looper设置一个Printer，打点统计dispatchMessage方法执行的时间，如果超出阀值，表示发生卡顿，则dump出各种信息，提供开发者分析性能瓶颈。

缺点：

* 无法准确判断是否真正出现ANR，只能说明APP发生了UI阻塞，需要进行二次校验。
* 无法得到完整ANR日志
* 隶属于卡顿优化的方式

3.[**ANR-WatchDog**](https://github.com/SalomonBrys/ANR-WatchDog)

ANR-WatchDog是一种非侵入式的ANR监控组件，可以用于**线上ANR的监控**，ANR-WatchDog使用方式如下：

在项目的app/build.gradle中添加依赖：

```
implementation 'com.github.anrwatchdog:anrwatchdog:1.4.0'
```

然后，在应用的`Application`的`onCreate`方法中添加如下代码启动ANR-WatchDog：

```
new ANRWatchDog().start();
```

ANRWatchDog的使用与实现都非常简单，整个库只有两个类，一个是ANRWatchDog，另一个是ANRError。

[ANRWatchDog](https://github.com/SalomonBrys/ANR-WatchDog/blob/master/anr-watchdog/src/main/java/com/github/anrwatchdog/ANRWatchDog.java)的实现解析如下：

```
/**
* A watchdog timer thread that detects when the UI thread has frozen.
*/
public class ANRWatchDog extends Thread {
```

ANRWatchDog继承Thread类，也就是它是一个线程，其run方法如下所示：

```
private static final int DEFAULT_ANR_TIMEOUT = 5000;

private volatile long _tick = 0;
private volatile boolean _reported = false;

private final Runnable _ticker = new Runnable() {
    @Override public void run() {
        _tick = 0;
        _reported = false;
    }
};

@Override
public void run() {
    // 1、首先，将线程命名为|ANR-WatchDog|。
    setName("|ANR-WatchDog|");

    // 2、接着，声明了一个默认的超时间隔时间，默认的值为5000ms。
    long interval = _timeoutInterval;
    // 3、然后，在while循环中通过_uiHandler去post一个_ticker Runnable。
    while (!isInterrupted()) {
        // 3.1 这里的_tick默认是0，所以needPost即为true。
        boolean needPost = _tick == 0;
        // 这里的_tick加上了默认的5000ms
        _tick += interval;
        if (needPost) {
            _uiHandler.post(_ticker);
        }

        // 接下来，线程会sleep一段时间，默认值为5000ms。
        try {
            Thread.sleep(interval);
        } catch (InterruptedException e) {
            _interruptionListener.onInterrupted(e);
            return ;
        }

        // 4、如果主线程没有处理Runnable，即_tick的值没有被赋值为0，则说明发生了ANR，第二个_reported标志位是为了避免重复报道已经处理过的ANR。
        if (_tick != 0 && !_reported) {
            //noinspection ConstantConditions
            if (!_ignoreDebugger && (Debug.isDebuggerConnected() || Debug.waitingForDebugger())) {
                Log.w("ANRWatchdog", "An ANR was detected but ignored because the debugger is connected (you can prevent this with setIgnoreDebugger(true))");
                _reported = true;
                continue ;
            }

            interval = _anrInterceptor.intercept(_tick);
            if (interval > 0) {
                continue;
            }

            final ANRError error;
            if (_namePrefix != null) {
                error = ANRError.New(_tick, _namePrefix, _logThreadsWithoutStackTrace);
            } else {
                // 5、如果没有主动给ANR_Watchdog设置线程名，则会默认会使用ANRError的NewMainOnly方法去处理ANR。
                error = ANRError.NewMainOnly(_tick);
            }
           
           // 6、最后会通过ANRListener调用它的onAppNotResponding方法，其默认的处理会直接抛出当前的ANRError，导致程序崩溃。 _anrListener.onAppNotResponding(error);
            interval = _timeoutInterval;
            _reported = true;
        }
    }
}
```

注释1：将线程命名为了|ANR-WatchDog|。注释2：声明了一个默认的超时间隔时间，默认的值为5000ms。注释3：在while循环中通过\_uiHandler去post一个\_ticker Runnable。注意这里的\_tick默认是0，所以needPost即为true。接下来，线程会sleep一段时间，默认值为5000ms。注释4：**如果主线程没有处理Runnable，即\_tick的值没有被赋值为0，则说明发生了ANR，第二个\_reported标志位是为了避免重复报道已经处理过的ANR。如果发生了ANR**，就会调用接下来的代码，开始会处理debug的情况，然后，我们看到注释5处，如果没有主动给ANR\_Watchdog设置线程名，则会默认会使用ANRError的NewMainOnly方法去处理ANR。

[ANRError](https://github.com/SalomonBrys/ANR-WatchDog/blob/master/anr-watchdog/src/main/java/com/github/anrwatchdog/ANRError.java)的NewMainOnly方法如下所示：

```
/**
 * The minimum duration, in ms, for which the main thread has been blocked. May be more.
 */
public final long duration;

static ANRError NewMainOnly(long duration) {
    // 1、获取主线程的堆栈信息
    final Thread mainThread = Looper.getMainLooper().getThread();
    final StackTraceElement[] mainStackTrace = mainThread.getStackTrace();

    // 2、返回一个包含主线程名、主线程堆栈信息以及发生ANR的最小时间值的实例。
    return new ANRError(new $(getThreadTitle(mainThread), mainStackTrace).new _Thread(null), duration);
}
复制代码
```

注释1：首先获了主线程的堆栈信息，然后返回了一个包含主线程名、主线程堆栈信息以及发生ANR的最小时间值的实例。（我们可以**改造其源码在此时添加更多的卡顿现场信息，如CPU 使用率和调度信息、内存相关信息、I/O 和网络相关的信息等等**）

接下来，我们再回到ANRWatchDog的run方法中的注释6处，最后这里会通过ANRListener调用它的onAppNotResponding方法，其默认的处理会直接抛出当前的ANRError，导致程序崩溃。对应的代码如下所示：

```
private static final ANRListener DEFAULT_ANR_LISTENER = new ANRListener() {
    @Override public void onAppNotResponding(ANRError error) {
        throw error;
    }
};
```

了解了ANRWatchDog的实现原理之后，我们试一试它的效果如何。首先，我们给MainActivity中的悬浮按钮添加主线程休眠10s的代码，如下所示：

```
@OnClick({R.id.main_floating_action_btn})
void onClick(View view) {
    switch (view.getId()) {
        case R.id.main_floating_action_btn:
            try {
                // 对应项目中的第170行
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            jumpToTheTop();
            break;
        default:
            break;
    }
}
```

然后，我们重新安装运行项目，点击悬浮按钮，发现在10s内都不能触发屏幕点击和触摸事件，并且在10s之后，应用直接发生了崩溃。接着，我们在Logcat过滤栏中输入**fatal**关键字，找出致命的错误，log如下所示：

```
2020-01-18 09:55:53.459 29924-29969/? E/AndroidRuntime: FATAL EXCEPTION: |ANR-WatchDog|
Process: json.chao.com.wanandroid, PID: 29924
com.github.anrwatchdog.ANRError: Application Not Responding for at least 5000 ms.
Caused by: com.github.anrwatchdog.ANRError?$_Thread: main (state = TIMED_WAITING)
    at java.lang.Thread.sleep(Native Method)
    at java.lang.Thread.sleep(Thread.java:373)
    at java.lang.Thread.sleep(Thread.java:314)
    // 1
    at json.chao.com.wanandroid.ui.main.activity.MainActivity.onClick(MainActivity.java:170)
    at json.chao.com.wanandroid.ui.main.activity.MainActivity_ViewBinding$1.doClick(MainActivity_ViewBinding.java:45)
    at butterknife.internal.DebouncingOnClickListener.onClick(DebouncingOnClickListener.java:22)
    at android.view.View.performClick(View.java:6311)
    at android.view.View$PerformClick.run(View.java:24833)
    at android.os.Handler.handleCallback(Handler.java:794)
    at android.os.Handler.dispatchMessage(Handler.java:99)
    at android.os.Looper.loop(Looper.java:173)
    at android.app.ActivityThread.main(ActivityThread.java:6653)
    at java.lang.reflect.Method.invoke(Native Method)
    at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:547)
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:821)
 Caused by: com.github.anrwatchdog.ANRError?$_Thread: AndroidFileLogger./storage/emulated/0/Android/data/json.chao.com.wanandroid/log/ (state = RUNNABLE)
 
 
复制代码
```

可以看到，发生崩溃的线程正是|ANR-WatchDog|。我们重点关注注释1，这里发生崩溃的位置是在MainActivity的onClick方法，对应的行数为170行，从前可知，这里正是线程休眠的地方。

接下来，我们来分析一下ANR-WatchDog的实现原理。

**ANR-WatchDog原理：**

* 首先，我们调用了ANR-WatchDog的start方法，然后这个线程就会开始工作。
* 然后，我们**通过主线程的Handler post一个消息将主线程的某个值进行一个加值的操作**。
* post完成之后呢，我们这个线程就sleep一段时间。
* **在sleep之后呢，它就会来检测我们这个值有没有被修改，如果这个值被修改了，那就说明我们在主线程中执行了这个message，即表明主线程没有发生卡顿，否则，则说明主线程发生了卡顿**。
* 最后，ANR-WatchDog就会判断发生了ANR，抛出一个异常给我们。

最后，ANR-WatchDog的工作流程简图如下所示：

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/080a3bec391d4e85b5215ed37791c132~tplv-k3u1fbpfcp-zoom-1.image)

上面我们最后说到，如果检测到主线程发生了卡顿，则会抛出一个ANR异常，这将会导致应用崩溃，显然不能将这种方案带到线上，那么，**有什么方式能够自定义最后发生卡顿时的处理过程吗？**

其实ANR-WatchDog自身就实现了一个我们自身也可以去实现的**ANRListener，通过它，我们就可以对ANR事件去做一个自定义的处理**，比如将堆栈信息压缩后保存到本地，并在适当的时间上传到APM后台。

4.[**AndroidPerformanceMonitor**](https://github.com/markzhai/AndroidPerformanceMonitor)

![](<../../.gitbook/assets/image (424).png>)



### 6. 如何避免ANR

1. 主线程尽量只做UI相关的操作，避免耗时操作，比如过度复杂的UI绘制，网络操作，文件IO操作，序列化等；
2. 避免主线程跟工作线程发生锁的竞争，减少系统耗时binder的调用，谨慎使用sharePreference，注意主线程执行provider query操作。

### 7. 典型案例

#### 主线程耗时操作:

```
"main" prio=5 tid=1 Runnable
  | group="main" sCount=0 dsCount=0 flags=0 obj=0x72c1f1f0 self=0xf0d37800
  | sysTid=13206 nice=-10 cgrp=default sched=0/0 handle=0xf12f4dc8
  | state=R schedstat=( 21847578037 1856772929 7897 ) utm=2139 stm=45 core=3 HZ=100
  | stack=0xff784000-0xff786000 stackSize=8192KB
  | held mutexes= "mutator lock"(shared held)  
  at com.oneplus.anr.MainActivity.InfiniteLoop(MainActivity.java:31) <<< 耗时堆栈
  at com.oneplus.anr.MainActivity.access$400(MainActivity.java:14)
  at com.oneplus.anr.MainActivity$6.onClick(MainActivity.java:132)
  at android.view.View.performClick(View.java:7125)
  at android.view.View.performClickInternal(View.java:7102)
  at android.view.View.access$3500(View.java:801)
  at android.view.View$PerformClick.run(View.java:27336)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7356)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
```

部分主线程耗时操作可以通过trace堆栈信息直接确认，如果能直接确认，根据上述**定位与分析方法**进行定位。

#### 死锁：

```
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72c1f1f0 self=0xf0d37800
  | sysTid=6304 nice=-10 cgrp=default sched=0/0 handle=0xf12f4dc8
  | state=S schedstat=( 612213735 63022150 522 ) utm=11 stm=50 core=0 HZ=100
  | stack=0xff784000-0xff786000 stackSize=8192KB
  | held mutexes=
  at com.oneplus.anr.MainActivity$1.run(MainActivity.java:56)
  - waiting to lock <0x0a082e95> (a java.lang.Object) held by thread 3
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7356)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
  
  "APP: Locker" prio=5 tid=3 Sleeping
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12c42090 self=0xda15d600
  | sysTid=6386 nice=0 cgrp=default sched=0/0 handle=0xc1667230
  | state=S schedstat=( 706530 0 2 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xc1564000-0xc1566000 stackSize=1040KB
  | held mutexes=
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x016c9876> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:440)
  - locked <0x016c9876> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:356)
  at com.oneplus.anr.MainActivity.Sleep(MainActivity.java:20)
  at com.oneplus.anr.MainActivity.access$100(MainActivity.java:14)
  at com.oneplus.anr.MainActivity$LockerThread.run(MainActivity.java:46)
  - locked <0x0a082e95> (a java.lang.Object)
```

**Binder通信失败：**

```
09:56:20.888 26662 27432 E IPCThreadState: binder thread pool (15 threads) starved for 601 ms
```

### 8. 参考资料

* [彻底理解安卓应用无响应机制](https://zhuanlan.zhihu.com/p/62509258)
* [理解Android ANR的信息收集过程](http://gityuan.com/2016/12/02/app-not-response/)
* [Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)
* [理解Android ANR的触发原理](http://gityuan.com/2016/07/02/android-anr/)
* [今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://juejin.cn/post/6940061649348853796)
* [今日头条 ANR 优化实践系列 - 监控工具与分析思路](https://juejin.cn/post/6942665216781975582)
* [AndroidPerformanceMonitor](https://github.com/markzhai/AndroidPerformanceMonitor)
* [ANR-WatchDog](https://github.com/SalomonBrys/ANR-WatchDog)



