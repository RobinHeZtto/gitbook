# Leakcanary

**LeakCanary**，大名鼎鼎的`Square`公司开发的一款Android应用内存泄漏检测库，项目地址：[https://square.github.io/leakcanary/](https://square.github.io/leakcanary/)

![](<../../.gitbook/assets/image (247).png>)

### **1.** ReferenceQueue

#### **Reference:**

Reference是SoftReference，WeakReference，PhantomReference的父类，

![](<../../.gitbook/assets/image (16).png>)

查看源码，发现Reference是个链表节点

```
public abstract class Reference<T> {
  final ReferenceQueue<? super T> queue;
  Reference queueNext;
  Reference<?> pendingNext;
}
```

1. queueNext 指向入队后的下一个引用
2. pendingNext 指向即将入队(unenqueued)的下一个引用
3. queue 指向构造函数传入的 ReferenceQueue

queueNext 和 pendingNext 均是 Reference 类型，queue 为 ReferenceQueue 类型

#### ReferenceQueue：

如果一个对象被回收了，比如是 WeakReference 方式，且创建时传入了 ReferenceQueue，那么此 WeakReference 会最终进入到其对应的 ReferenceQueue 中。LeakCanary 最本质的原理就是这样，检测 ReferenceQueue 有无此 Reference，有则没有泄露，应该还需要自行 remove。没有的话主动调用 Rumtime.gc() 之后再来观察队列，依然没有则判定存在了内存泄漏。

![](<../../.gitbook/assets/image (8).png>)

### 2. LeakCanary2 <a href="#leakcanary" id="leakcanary"></a>

2019 年 11 月 ，LeakCanary2 正式版发布，和 LeakCanary1 相比，LeakCanary2 有以下改动：

* 完全使用 Kotlin 重写
* 使用新的 Heap 分析工具 [Shark](https://square.github.io/leakcanary/shark/)替换之前的 [haha](https://github.com/square/haha)，按官方的说法，内存占用减少了 10 倍
* 泄露类型分组

其中，将 Heap 分析模块作为一个独立的模块，这意味着，可以基于 Shark 来做很多有意思的事情，比如，用于线上分析或者开发一个”自己”的 LeakCanary。

#### 使用方式：

`LeakCanary1.x`的使用方式，首先在`build.grdle` 中配置依赖

```
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
  releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
}
```

接着在 Application 中调用 `LeakCanary.install()` 方法。而 LeakCanary2 集成只需要增加以下依赖即可：

```
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.2'
}
```

#### 整体架构 <a href="#zheng-ti-jia-gou" id="zheng-ti-jia-gou"></a>

在分析源码之前，先看一下LeakCanary 的整体结构，这有助于我们对项目整体设计上有一定理解。LeakCanary2 有以下几个模块：

*   **leakcanary-android**

    集成入口模块，提供 LeakCanary 安装，公开 API 等能力
*   **leakcanary-android-core**

    核心模块
*   **leakcanary-android-instrumentation**

    用于 Android Test 的模块
*   **leakcanary-android-process**

    和 leakcanary-android 一样，区别是会在单独的进程进行分析
*   **leakcanary-object-watcher-android，leakcanary-object-watcher-android-androidx，leakcanary-watcher-android-support-fragments**

    对象实例观察模块，在 Activity，Fragment 等对象的生命周期中，注册对指定对象实例的观察，有 Activity，Fragment，Fragment View，ViewModel 等
*   **shark-android**

    提供特定于 Android 平台的分析能力。例如设备的信息，Android 版本，已知的内存泄露问题等
*   **shark，shark-test**

    hprof 文件解析与分析的入口模块，还有对应的 Test 模块
*   **shark-graph**

    分析堆中对象的关系图模块
*   **shark-hprof，shark-hprof-test**

    解析 hprof 文件模块，还有对应的 Test 模块
*   **shark-log**

    日志模块
*   **shark-cli**

    shark-android 的 cli 版本

### 3. 源码解析

#### AppWatcherInstaller

Leakcancary以ContentProvider的形式自动注入到宿主工程中，`ContentProvider#Oncreate()`在`Application#attachBaseContext` 与`Appliction#onCreate之间调用`

```
internal sealed class AppWatcherInstaller : ContentProvider() {

  internal class MainProcess : AppWatcherInstaller()

  internal class LeakCanaryProcess : AppWatcherInstaller()

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }

  ......
}
```

`AppWatcherInstaller` 有两个实现类，一个是 `MainProcess`，当我们使用 leakcanary-android 模块时，会默认使用这个，表示在当前 App 进程中使用 LeakCanary。另外一个类为 `LeakCanaryProcess`，当使用 leakcanary-android-process 模块代替 leakcanary-android 模块时，则会使用这个类。

#### AppWatcher

`AppWatcher#manualInstall`主要完成`Activity、Fragment、ViewModel`等对象的注册与观察

```
object AppWatcher {

  @JvmOverloads
  fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
  ) {
    // 检查是否在主线程
    checkMainThread()
    // 检查是否已经install
    check(!isInstalled) {
      "AppWatcher already installed"
    }
    // 检查retainedDelayMillis有效性
    check(retainedDelayMillis >= 0) {
      "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    this.retainedDelayMillis = retainedDelayMillis
    if (application.isDebuggableBuild) {
      LogcatSharkLog.install()
    }
    // InternalLeakCanary初始化
    LeakCanaryDelegate.loadLeakCanary(application)

    // install所有需要观察的对象
    watchersToInstall.forEach {
      it.install()
    }
  }
}
```

#### InternalLeakCanary

`AppWatcher#manualInstall`主要过程中，通过`LeakCanaryDelegate.loadLeakCanary`完成InternalLeakCanary的初始化

```
internal object LeakCanaryDelegate {

  @Suppress("UNCHECKED_CAST")
  // 属性委托
  val loadLeakCanary by lazy {
    try {
      val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
      // 通过反射，获取（Application) -> Unit形式的函数对象
      leakCanaryListener.getDeclaredField("INSTANCE")
        .get(null) as (Application) -> Unit
    } catch (ignored: Throwable) {
      NoLeakCanary
    }
  }

  object NoLeakCanary : (Application) -> Unit, OnObjectRetainedListener {
    override fun invoke(application: Application) {
    }

    override fun onObjectRetained() {
    }
  }
}
```

当调用`loadLeakCanary`时将执行`InternalLeakCanary#invoke`方法

```
  override fun invoke(application: Application) {
    _application = application

    // 如果当前APP处于debug模式，不处理
    checkRunningInDebuggableBuild()

    // 在AppWatcher.objectWatcher中添加OnObjectRetainedListener
    AppWatcher.objectWatcher.addOnObjectRetainedListener(this)

    // heapdumper
    val heapDumper = AndroidHeapDumper(application, createLeakDirectoryProvider(application))

    // gc
    val gcTrigger = GcTrigger.Default

    val configProvider = { LeakCanary.config }

    // 创建heapdump线程及对象
    val handlerThread = HandlerThread(LEAK_CANARY_THREAD_NAME)
    handlerThread.start()
    val backgroundHandler = Handler(handlerThread.looper)

    heapDumpTrigger = HeapDumpTrigger(
      application, backgroundHandler, AppWatcher.objectWatcher, gcTrigger, heapDumper,
      configProvider
    )
    application.registerVisibilityListener { applicationVisible ->
      this.applicationVisible = applicationVisible
      heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
    }
    registerResumedActivityListener(application)
    addDynamicShortcut(application)

    // We post so that the log happens after Application.onCreate()
    mainHandler.post {
      // https://github.com/square/leakcanary/issues/1981
      // We post to a background handler because HeapDumpControl.iCanHasHeap() checks a shared pref
      // which blocks until loaded and that creates a StrictMode violation.
      backgroundHandler.post {
        SharkLog.d {
          when (val iCanHasHeap = HeapDumpControl.iCanHasHeap()) {
            is Yup -> application.getString(R.string.leak_canary_heap_dump_enabled_text)
            is Nope -> application.getString(
              R.string.leak_canary_heap_dump_disabled_text, iCanHasHeap.reason()
            )
          }
        }
      }
    }
  }
```

可以看到`InternalLeakCanary#invoke`中主要完成了AppWatcher.objectWatcher的监听，gcTrigger、heapDumpTrigger等对象的创建。

继续回到`AppWatcher#manualInstall`中，最后完成了所有需观察对象的install

```
  @JvmOverloads
  fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
  ) {
    ....

    // install所有需要观察的对象
    watchersToInstall.forEach {
      it.install()
    }
  }
}


fun appDefaultWatchers(
  application: Application,
  reachabilityWatcher: ReachabilityWatcher = objectWatcher
): List<InstallableWatcher> {
  return listOf(
    ActivityWatcher(application, reachabilityWatcher),
    FragmentAndViewModelWatcher(application, reachabilityWatcher),
    RootViewWatcher(reachabilityWatcher),
    ServiceWatcher(reachabilityWatcher)
  )
}
```

所有都wathcer都包含一个ReachabilityWatcher，用于检测retained objects

```
  val objectWatcher = ObjectWatcher(
    clock = { SystemClock.uptimeMillis() },
    checkRetainedExecutor = {
      check(isInstalled) {
        "AppWatcher not installed"
      }
      // 在主线程中post retainedDelayMillis后执行
      mainHandler.postDelayed(it, retainedDelayMillis)
    },
    isEnabled = { true }
  )
```

#### ActivityWatcher

```
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      // 在onActivityDestroyed之后调用expectWeaklyReachable加入观察
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}
```

`ActivityWatcher#install`主要实现即注册activity生命周期回掉，并在`onActivityDestroyed`以后调用`reachabilityWatcher.expectWeaklyReachable`加入观察

#### ObjectWatcher

```
  private val queue = ReferenceQueue<Any>()
  
  @Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  // 从观察列表中移除已经销毁的对象
  removeWeaklyReachableObjects()
  // 生成随机字符串作为观察对象的key
  val key = UUID.randomUUID()
    .toString()
  val watchUptimeMillis = clock.uptimeMillis()
  // 创建对应的观察对象的弱引用
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  SharkLog.d {
    "Watching " +
      (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
      (if (description.isNotEmpty()) " ($description)" else "") +
      " with key $key"
  }
  // 加入观察队列
  watchedObjects[key] = reference
  // post到主线程中延时retainedDelayMillis处理
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}
  
  
private fun removeWeaklyReachableObjects() {
// WeakReferences are enqueued as soon as the object to which they point to becomes weakly
// reachable. This is before finalization or garbage collection has actually happened.
var ref: KeyedWeakReference?
do {
    // queue中存在，代表已经被回收
    ref = queue.poll() as KeyedWeakReference?
  if (ref != null) {
    // 从观察对象中移除已经被回收的对象
    watchedObjects.remove(ref.key)
  }
} while (ref != null)
}
```

expectWeaklyReachable中主要完成的操作是，创建观察对象对应的弱引用对象，并加入到观察队列，并延时retainedDelayMillis在主线程中执行moveToRetained

```
@Synchronized private fun moveToRetained(key: String) {
  // 再次从观察列表中移除已经销毁的对象
  removeWeaklyReachableObjects()
  val retainedRef = watchedObjects[key]
  // 对应的对象还在，没有被销毁，则回掉onObjectRetainedListeners方法
  if (retainedRef != null) {
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}
```

retainedDelayMillis时间后，观察的对象还没被销毁，回调`onObjectRetainedListeners`，上面`InternalLeakCanary#invoke`中注册了`onObjectRetainedListeners`，这里将进入到`InternalLeakCanary#onObjectRetained`执行

#### InternalLeakCanary

```
  override fun onObjectRetained() = scheduleRetainedObjectCheck()

  fun scheduleRetainedObjectCheck() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.scheduleRetainedObjectCheck()
    }
  }
```

scheduleRetainedObjectCheck中调用`heapDumpTrigger.scheduleRetainedObjectCheck`执行dumpheap操作

#### HeapDumpTrigger

```
  fun scheduleRetainedObjectCheck(
    delayMillis: Long = 0L
  ) {
    // 同时只会有一个任务
    val checkCurrentlyScheduledAt = checkScheduledAt
    if (checkCurrentlyScheduledAt > 0) {
      return
    }
    checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
    // post到子线程中执行
    backgroundHandler.postDelayed({
      checkScheduledAt = 0
      checkRetainedObjects()
    }, delayMillis)
  }
  
  private fun checkRetainedObjects() {
  val iCanHasHeap = HeapDumpControl.iCanHasHeap()

  val config = configProvider()

  if (iCanHasHeap is Nope) {
    if (iCanHasHeap is NotifyingNope) {
      // Before notifying that we can't dump heap, let's check if we still have retained object.
      var retainedReferenceCount = objectWatcher.retainedObjectCount

      if (retainedReferenceCount > 0) {
        // 先执行一次gc
        gcTrigger.runGc()
        retainedReferenceCount = objectWatcher.retainedObjectCount
      }

      // 检测当前保留对象数量
      val nopeReason = iCanHasHeap.reason()
      val wouldDump = !checkRetainedCount(
        retainedReferenceCount, config.retainedVisibleThreshold, nopeReason
      )

      if (wouldDump) {
        val uppercaseReason = nopeReason[0].toUpperCase() + nopeReason.substring(1)
        onRetainInstanceListener.onEvent(DumpingDisabled(uppercaseReason))
        showRetainedCountNotification(
          objectCount = retainedReferenceCount,
          contentText = uppercaseReason
        )
      }
    } else {
      SharkLog.d {
        application.getString(
          R.string.leak_canary_heap_dump_disabled_text, iCanHasHeap.reason()
        )
      }
    }
    return
  }

  var retainedReferenceCount = objectWatcher.retainedObjectCount

  if (retainedReferenceCount > 0) {
    // 再次执行gc
    gcTrigger.runGc()
    retainedReferenceCount = objectWatcher.retainedObjectCount
  }

  if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

  val now = SystemClock.uptimeMillis()
  val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
  if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
    onRetainInstanceListener.onEvent(DumpHappenedRecently)
    showRetainedCountNotification(
      objectCount = retainedReferenceCount,
      contentText = application.getString(R.string.leak_canary_notification_retained_dump_wait)
    )
    scheduleRetainedObjectCheck(
      delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
    )
    return
  }

  dismissRetainedCountNotification()
  val visibility = if (applicationVisible) "visible" else "not visible"
  // 执行 dump heap
  dumpHeap(
    retainedReferenceCount = retainedReferenceCount,
    retry = true,
    reason = "$retainedReferenceCount retained objects, app is $visibility"
  )
}
```

#### shark <a href="#shark" id="shark"></a>

此前LeakCanary用于分析hprof的工具依次包括HAHA、HAHA2以及perflib。shark是square团队开发的一款全新的分析hprof文件的工具，其官方宣布比Android Studio用于memory profiler的核心库`perflib` 要快8倍并且内存占用少10倍，更加适合手机端的分析工具。其目的就是提供快速解析hprof文件和分析快照的能力，并找出真正的泄漏对象以及对象到GcRoot的最短引用路径链，以便帮助开发者更加直观的找出泄漏的真正原因。

> 由此可知，先前初步分析得到的泄漏对象与这里的堆分析和得到的真正泄漏对象并没有依赖关系，先前的初步分析仅仅只是一个触发堆转储和分析操作的过滤。

hprof文件的标准协议主要由head和body组成，head包含一些元信息，例如文件协议的版本、开始的时间戳等。body则是由一系列不同类型的Record组成，Record主要用于描述trace、object（实例 / 数组 / 类 / …）、thread等信息，依次分为4个部分：TAG、TIMESTAMP、LENGTH、BODY，其中TAG就是表示Record类型，LENGTH用于指示BODY的长度。Record之间依次排列或嵌套，最终组成hprof文件。

在Shark库的设计中，`Hprof`类就是用于描述从内存中dump出的`.hprof`文件；密封类`HprofRecord`的各个子类一一对应上述协议中的各种Record；真正的解析则由`HprofReader`实现，核心方法为`readHprofRecords`()：

```
fun readHprofRecords(
    recordTypes: Set<KClass<out HprofRecord>>, // 关心的record类型
    listener: OnHprofRecordListener // 回传解析得到的record映射对象
  ) {
    ...
    while (!exhausted()) { // 循环读取标签，直至结束
	  ...
      when (tag) {
          STRING_IN_UTF8 -> {...}
          ...
          HEAP_DUMP, HEAP_DUMP_SEGMENT -> {
              val heapDumpTag = readUnsignedByte()
              when (heapDumpTag) {
                  ...
                  ROOT_THREAD_OBJECT -> {
                    if (readGcRootRecord) {
                      val recordPosition = position
                      val gcRootRecord = GcRootRecord(
                          gcRoot = ThreadObject(
                              id = readId(),
                              threadSerialNumber = readInt(),
                              stackTraceSerialNumber = readInt()
                          )
                      )
                      listener.onHprofRecord(recordPosition, gcRootRecord)
                    } else {
                      skip(identifierByteSize + intByteSize + intByteSize)
                    }
                  }
                  ...
              }
              ...
          }
          ...
      }
    }
```

不难看出，readHprofRecords就是一个遵从协议实现的通用提取器，Record的类型有很多，通过指定关心的record类型和回调接口就可以将hprof文件转化为Record对象集合。

解析得到的records被进一步抽象为`HprofMemoryIndex`，Index的作用就是就是将得到的record按类型进行归类和计数，并通过特定规则进行排序；最终Index和Hprof一起再组成`HprofGraph`，graph做为hprof的最上层描述，将所有堆中数据抽象为了 `gcRoots、objects、classes、instances` 4种集合，并提供了快速定位dump堆中具体对象的能力。

#### HeapAnalyzer <a href="#heapanalyzer" id="heapanalyzer"></a>

heapAnalyzer作为shark模块的出入口，衔接解析和分析两大流程，核心逻辑如下：

```
Hprof.open(heapDumpFile)
    .use { hprof ->
        val graph = HprofHeapGraph.indexHprof(hprof, proguardMapping)
		...
        val findLeakInput = FindLeakInput(graph, leakFinders, referenceMatchers, computeRetainedHeapSize, objectInspectors)
        val (applicationLeaks, libraryLeaks) = findLeakInput.findLeaks()
        ...
        return HeapAnalysisSuccess(
            heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime), metadata,
            applicationLeaks, libraryLeaks
        )
    }
```

整个流程非常清晰：首先通过indexHprof解析得到graph，然后构造 `FindLeakInput` 开始分析并找出泄漏，最后包装分析结果返回。\
graph的组成和功能我们已经清楚了，我们直接来看`findLeaks`具体是如何工作的。整个流程分为了三个步骤，分别对应三个关键方法：`findRetainedObjects`、`findPathsFromGcRoots`、`buildLeakTraces`。

**findRetainedObjects**

顾名思义就是找到存活（泄漏）的所有对象，具体逻辑就是遍历由graph提供的objects集合，通过特定规则进行标记过滤，最后得到一组由存活对象id构成的集合。\
这里的特定规则由多个`ObjectInspector` 组成，根据LeakCanary的文档描述，默认的Inspector是建立在AOSP / libraries / JDK / …等基础上的开发经验和知识总结而来的，即对于Android而言（具体类`AndroidObjectInspectors`），我们知道哪些地方是可以造成内存泄漏的，因此只需要在内存快照中依次遍历特定的对象查看其是否 **满足 / 肯定不满足** 泄漏条件即可知道具体的泄漏对象，这些特定对象包括：

* android.widget.View间接引用的Activity的mDestroyed字段；
* android.widget.Editor内部的mTextView字段；
* android.app.Activity的mDestoryed字段；
* android.content.ContextWrapper间接引用的Activity的mDestroyed字段；
* android.app.Dialog的mDecor字段；
* Fragment的mFragmentManager字段；
* android.os.MessageQueue的mQuitting字段；
* android.view.ViewRootImpl的mView字段；
* android.view.Window的mDestroyed字段；
* android.widget.Toast中的mTN对象内部的mWM字段和mView字段；

对于JDK而言（具体类`ObjectInspectors`），我们知道哪些地方肯定不会泄漏，例如：ClassLoader、Class、匿名Class等，以及对于单例而言（具体类`AppSingletonInspector`），对象肯定不属于泄漏。

上述规则其实不难理解，内存泄漏的大致定义是：\
**无效的对象仍然与GcRoot保持了可达的引用路径，导致其无法被GC释放。**\
而「无效」是逻辑上的概念，GC是无法理解的，但作为开发人员是知道的，例如在Android平台上，应用具体的场景切换中我们是知道哪些对象的状态会转变为无效，例如销毁的Activity，因此才会在初始化时对其进行watch。

**findPathsFromGcRoots**

接下来就是找出所有从泄漏对象到GcRoot的引用路径链。总体思路是采用 **广度优先遍历** 算法从Roots向下查找，直到到达泄漏对象。由于涉及的代码繁多，这里仅阐述主要逻辑，整个过程是先从graph中得到的gcRoots被抽象为node按照优先级依次入队，作为队列的初始数据构成树根，然后开始从根节点开始遍历，对每个节点依据类型的不同采取对应的模式进行访问并得到所引用的对象集合，引用集合继续被抽象为node加入队列构成子节点以待后续遍历，如此反复直到当前遍历的node对象属于第一步中的泄漏对象，期间所经过的所有node便构成了从泄漏对象到GcRoot的引用路径。

上述过程中，用于维护遍历顺序的队列是通过toVisist、toVisitLast两对Queue和Set来构造的，其中Set用于保证树的遍历顺序始终遵从“从上至下”，使最终引用链结果不会包含更长的路径；两种Queue则提供一个优先级分级的概念，用于保证GcRoots中的ThreadObject优先于JavaFrame被遍历（当然也用于降低遍历LibraryLeak类型的泄漏的优先级），这样也是为了得到离泄漏对象最近的引用路径，避免得到形如thread-…->frame-…>thread-…>obj这样的结果。

除此之外，针对已知泄露（例如Android Framework内部的一些已知泄漏）的剔除也是在这里实现的，避免检测过程中反复打扰。已知泄漏使用`ReferenceMatcher`来描述，泄漏的具体引用使用`ReferencePattern`描述，分为threadLocal-variable、static-field、instance-field和native-globalVariable四种类型，并在遍历入队过程中，根据对象的类型不同，针对HeapClass、HeapInstance、HeapObjectArray提取对应的name在相应的已知泄漏集合中进行判断，以决定是否剔除。

当然还包括一些其它细节，例如在遍历入队过程中对基本类型和String等对象的剔除，都是为了得到更短更低重复性的路径集合。

**buildLeakTraces**

我们知道，一个对象被多个对象引用是很常见的行为，前面得到的引用路径集合泄漏对象和GcRoot之间可能存在多条路径，为了更利于分析，还需要进行裁剪。\
裁剪的过程并不复杂，首先先将路径链反转为从GcRoot到泄漏对象的方向，然后通过`updateTrie`方法转化为一个以无效node为根节点的树，最后再通过 **深度优先遍历** 算法得到从根节点（无效node的children）到叶子节点的所有路径，即为最终的最短路径。其中裁剪的逻辑就发生在构造树的过程中：

```
private fun deduplicateShortestPaths(inputPathResults: List<ReferencePathNode>): List<ReferencePathNode> {
    val rootTrieNode = ParentNode(0)
    for (pathNode in inputPathResults) {
        ...// 这里的path是已经反转后的路径
        updateTrie(pathNode, path, 0, rootTrieNode)
    }
    ...
}
private fun updateTrie(pathNode: ReferencePathNode, path: List<Long>, pathIndex: Int, parentNode: ParentNode) {
    val objectId = path[pathIndex]
    if (pathIndex == path.lastIndex) {
        // 1
        parentNode.children[objectId] = LeafNode(objectId, pathNode)
    } else {
        val childNode = parentNode.children[objectId] ?: {
            val newChildNode = ParentNode(objectId)
            parentNode.children[objectId] = newChildNode
            newChildNode
        }()
        if (childNode is ParentNode) {
            updateTrie(pathNode, path, pathIndex + 1, childNode)
        }
    }
}
```

方法`deduplicateShortestPaths`中的`rootTrieNode`作为树的根节点，其children指向所有的GcRoot；在注释1处，如果某条路径遍历到最后一个节点时发现起父节点parentNode的children中已经存在自身，即GcRoot和泄漏对象相同，说明此前已有一条相等或更长的重复路径，那么此处的覆盖就将更长的引用链丢弃（裁剪）了。

至此主要工作结束，剩余的逻辑主要是为引用路径链的每个节点赋予详细的描述信息，包括通过objectInspectors重新inspect等，最终构建为applicationLeaks和LibraryLeaks两种形式的泄露描述并得到HeapAnalysis，在Service中通过DefaultOnHeapAnalyzedListener写入数据库并Notification提示，这些后续逻辑相对简单，就不一一细说了。

### 4. 总结

LeakCanary的整个工作流程从向Activity/Fragment注册生命周期监听开始，当组件销毁时触发对对应对象的观察，利用WeakReference和手动触发GC双重机制来判断对象是否泄露，并在对象可能确实泄露时dump内存快照利用shark进行解析和分析，整个分析过程分为三个流程，先依据Android平台的特性和开发经验找出目标（泄露）对象，然后从GcRoot开始遍历，采用广度优先遍历算法得到所有符合条件的引用路径链，最终裁剪得到泄露对象到GcRoot的最短引用路径链，整个流程结束。

LeakCanary的实现逻辑还是非常清晰和高效的。LeakCanary也不是万能的，如何确定一个对象是否有效，它的规则并不能覆盖到所有的业务场景。
