# Android启动之SystemServer(下)

> 上篇已经分析了SystemServer进程的创建过程，在ZygoteInit.main() catch MethodAndArgsCaller异常后，将执行MethodAndArgsCaller.run并反射SystemServer.main()方法，下面将从SystemServer.main()开始分析。 相关的源代码在以下文件中：
>
> * frameworks/base/services/java/com/android/server/SystemServer.java
> * frameworks/base/core/java/android/app/ActivityThread.java
> * frameworks/base/core/java/android/app/ContextImpl.java

## SystemServer.main()

SystemServer.main()的代码如下：

```
<<< frameworks/base/services/java/com/android/server/SystemServer.java
​
/**
 * The main entry point from zygote.
 */
public static void main(String[] args) {
    new SystemServer().run();
}
​
public SystemServer() {
    // Check for factory test mode.
    mFactoryTestMode = FactoryTest.getMode();
}
```

main()中创建SystemServer对象并调用其run()方法，run()的实现如下：

```
<<< frameworks/base/services/java/com/android/server/SystemServer.java
​
// system_server进程binder线程池最大线程数
private static final int sMaxBinderThreads = 31;
​
private void run() {
    try {
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "InitBeforeStartServices");
        // 如果系统时间早于1970年，调整为1970年
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }
​
        // 设置language相关属性
        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();
​
            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }
​
        // Here we go!
        Slog.i(TAG, "Entered the Android system server!");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());
​
        // 设置persist.sys.dalvik.vm.lib.2属性值为当前虚拟机运行库，默认libart.so
        SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
​
        // 开启sampling profiler.
        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            mProfilerSnapshotTimer = new Timer();
            mProfilerSnapshotTimer.schedule(new TimerTask() {
                    @Override
                    public void run() {
                        SamplingProfilerIntegration.writeSnapshot("system_server", null);
                    }
                }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        }
​
        // 清除VM内存增长上限
        VMRuntime.getRuntime().clearGrowthLimit();
​
        // 设置VM堆内存利用率为0.8
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
​
        // Some devices rely on runtime fingerprint generation, so make sure
        // we've defined it before booting further.
        Build.ensureFingerprintProperty();
​
        // 设置需指定用户访问环境变量
        Environment.setUserRequired(true);
​
        // Within the system server, any incoming Bundles should be defused
        // to avoid throwing BadParcelableException.
        BaseBundle.setShouldDefuse(true);
​
        // 确保binder调用运行在前台优先级(foreground priority)
        BinderInternal.disableBackgroundScheduling(true);
​
        // 设置system_server binder线程池最大线程数为sMaxBinderThreads
        BinderInternal.setMaxThreads(sMaxBinderThreads);
​
        // 主线程的Looper在当前线程中循环
        android.os.Process.setThreadPriority(
            android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();
​
        // 加载libandroid_servers.so(对应源码frameworks/base/services/core/jni/)，初始化本地服务
        System.loadLibrary("android_servers");
​
        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();
​
        // 获取系统Context，后面具体分析
        createSystemContext();
​
        // 创建SystemServiceManager，用于启动系统服务
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        // 将SystemServiceManager也添加到LocalServices中管理
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
​
    // 启动服务
    try {
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
​
    // debug版本log输出到dropbox
    if (StrictMode.conditionallyEnableDebugLogging()) {
        Slog.i(TAG, "Enabled StrictMode for system server main thread.");
    }
​
    // 消息队列循环.
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

run()中主要完成相关的初始化工作，启动系统服务，随后进入到消息队列的循环中。下面将具体分析一下createSystemContext()与启动服务的过程。

## createSystemContext()

createSystemContext()的实现如下：

```
<<< frameworks/base/services/java/com/android/server/SystemServer.java
​
private void createSystemContext() {
    ActivityThread activityThread = ActivityThread.systemMain();
    mSystemContext = activityThread.getSystemContext();
    mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
}
```

createSystemContext()中创建了ActivityThread对象，并通过其getSystemContext获取系统Context，然后通过系统Context设置主题，接下来一步步分析此过程。

```
<<< frameworks/base/core/java/android/app/ActivityThread.java
​
public static ActivityThread systemMain() {
    // The system process on low-memory devices do not get to use hardware
    // accelerated drawing, since this can add too much overhead to the
    // process.
    if (!ActivityManager.isHighEndGfx()) {
        HardwareRenderer.disable(true);
    } else {
        HardwareRenderer.enableForegroundTrimming();
    }
    ActivityThread thread = new ActivityThread();
    thread.attach(true);
    return thread;
}
```

ActivityThread.systemMain()中创建ActivityThread对象并调用其attch方法，具体实现如下：

```
<<< frameworks/base/core/java/android/app/ActivityThread.java
​
ActivityThread() {
    mResourcesManager = ResourcesManager.getInstance();
}
​
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
    ......  
    } else {
        // 设置DDMS中名称
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId());
        try {
            // 创建Instrumentation对象   
            mInstrumentation = new Instrumentation();
            // 通过ContextImpl创建context对象
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            // 创建Application对象并调用onCreate()方法                    
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate Application():" + e.toString(), e);
        }
    }
​
    // add dropbox logging to libcore
    DropBox.setReporter(new DropBoxReporter());
​
    ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
        public void onConfigurationChanged(Configuration newConfig) {...}
        public void onLowMemory() {...}
        public void onTrimMemory(int level) {...}
    });
}
```

ActivityThread的attach方法中通过getSystemContext获取系统Context对象，getSystemContext的实现如下：

```
<<< frameworks/base/core/java/android/app/ActivityThread.java
​
public ContextImpl getSystemContext() {
    synchronized (this) {
        if (mSystemContext == null) {
            mSystemContext = ContextImpl.createSystemContext(this);
        }
        return mSystemContext;
    }
}
​
<<< frameworks/base/core/java/android/app/ContextImpl.java
​
static ContextImpl createSystemContext(ActivityThread mainThread) {
    LoadedApk packageInfo = new LoadedApk(mainThread);
    ContextImpl context = new ContextImpl(null, mainThread,
            packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
    context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
            context.mResourcesManager.getDisplayMetricsLocked());
    return context;
}
​
<<< frameworks/base/core/java/android/app/LoadedApk.java
​
LoadedApk(ActivityThread activityThread) {
    mActivityThread = activityThread;
    mApplicationInfo = new ApplicationInfo();
    // packageName设置成android
    mApplicationInfo.packageName = "android";
    mPackageName = "android";
    mAppDir = null;
    mResDir = null;
    mSplitAppDirs = null;
    mSplitResDirs = null;
    mOverlayDirs = null;
    mSharedLibraries = null;
    mDataDir = null;
    mDataDirFile = null;
    mLibDir = null;
    mBaseClassLoader = null;
    mSecurityViolation = false;
    mIncludeCode = true;
    mRegisterPackage = false;
    mClassLoader = ClassLoader.getSystemClassLoader();
    mResources = Resources.getSystem();
}
```

getSystemContext()中调用ContextImpl.createSystemContext()创建了ContextImpl对象，以获取类似apk的Context上下文环境，在创建ContextImpl同时也需要LoadedApk对象，对应framework-res.apk，PackageName命名为android。

## StartServices

SystemServer分成三类启动服务，分别时Bootstrap，Core，Other，首先分析startBootstrapServices启动Bootstrap服务流程。

```
private void startBootstrapServices() {
    // 等待与installd建立socket通信
    Installer installer = mSystemServiceManager.startService(Installer.class);
​
    // 启动ActivityManagerService
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
​
    // 启动PowerManagerService
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
​
    // 初始化PowerManagement
    mActivityManagerService.initPowerManagement();
​
    // 启动LightsService
    mSystemServiceManager.startService(LightsService.class);
​
    // 启动DisplayManagerService
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
​
    // Phase100阶段
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
​
    // 当设备加密时仅仅运行core
    String cryptState = SystemProperties.get("vold.decrypt");
    if (ENCRYPTING_STATE.equals(cryptState)) {
        Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
        mOnlyCore = true;
    } else if (ENCRYPTED_STATE.equals(cryptState)) {
        Slog.w(TAG, "Device encrypted - only parsing core apps");
        mOnlyCore = true;
    }
​
    // 启动PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();
​
    // Manages A/B OTA dexopting. This is a bootstrap service as we need it to rename
    // A/B artifacts after boot, before anything else might touch/need them.
    // Note: this isn't needed during decryption (we don't have /data anyways).
    if (!mOnlyCore) {
        boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt",
                false);
        if (!disableOtaDexopt) {
            traceBeginAndSlog("StartOtaDexOptService");
            try {
                OtaDexoptService.main(mSystemContext, mPackageManagerService);
            } catch (Throwable e) {
                reportWtf("starting OtaDexOptService", e);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
            }
        }
    }
​
    // 启动UserManagerService
    mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
​
    // Initialize attribute cache used to cache resources from packages.
    AttributeCache.init(mSystemContext);
​
    // Set up the Application instance for the system process and get started.
    mActivityManagerService.setSystemProcess();
​
    // 启动SensorService
    startSensorService();
}
```

startBootstrapServices()中主要启动了ActivityManagerService, PowerManagerService, LightsService, DisplayManagerService， PackageManagerService， UserManagerService， sensor服务。

```
private void startCoreServices() {
    // 启动BatteryService(Tracks the battery level.  Requires LightService)
    mSystemServiceManager.startService(BatteryService.class);
​
    // 启动UsageStatsService
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));
    // Update after UsageStatsService is available, needed before performBootDexOpt.
    mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();
​
    // 启动WebViewUpdateService
    mSystemServiceManager.startService(WebViewUpdateService.class);
}
```

startCoreServices()中主要启动了BatteryService，UsageStatsService，WebViewUpdateService。
