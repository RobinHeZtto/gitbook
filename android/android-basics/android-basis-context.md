# Android基础之Context全解析

## 1. 什么是Context

Android有意淡化进程的概念，在开发一个Android应用程序时，通常都不需要关心目标对象运行在哪个进程，只需要表明意图(Intent)，譬如拨打电话、查看图片、打开链接等；也不需要关心系统接口是在哪个进程实现的，只需要通过Context发起调用。对于一个Android应用程序而言，Context就像运行环境一样，无处不在。有了Context，一个Linux进程就摇身一变，成为了Android进程，变成了Android世界的公民，享有Android提供的各种服务。那么，一个Android应用程序需要一些什么服务呢？

* 获取应用资源，譬如：drawable、string、assets
* 操作四大组件，譬如：启动界面、发送广播、绑定服务、打开数据库
* 操作文件目录，譬如：获取/data/分区的数据目录、获取sdcard目录
* 检查授予权限，譬如：应用向外提供服务时，可以判定申请者是否具备访问权限
* 获取其他服务，有一些服务有专门的提供者，譬如：包管理服务、Activity管理服务、窗口管理服务

在应用程序中，随处都可访问这些服务，这些服务的访问入口就是Context，Android对Context类的说明是：

```
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
```

其意为：Context是一个抽象类，提供接口，用于访问应用程序运行所需服务，譬如启动Activity、发送广播、接受Intent等。Android是构建在Linux之上的，然而对于Android的应用开发者而言，已经不需要关心Linux进程的概念了，正是因为有了Context，为应用程序的运行提供了一个Android环境，开发者只需要关心Context提供了哪些接口。

ContextImpl, ContextWrapper, Activity, Service, Application这些都是Context的直接或间接子类, 他们的关系如下:

![](<../../.gitbook/assets/image (9).png>)

![](<../../.gitbook/assets/image (388).png>)

## 2. Application的创建过程

![](<../../.gitbook/assets/image (217).png>)

在Zygote创建应用进程后，将进入Java层的entry point执行，即ActivityThread.main()。

```
>>> android.app.ActivityThread

public static void main(String[] args) {
    ......
    Looper.prepareMainLooper();
    ......
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
    ......
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

ActivityThread.main中创建Looper，ActivityThread实例， 调用ActivityThread.attach后即进入Looper.loop循环中。

```
>>> android.app.ActivityThread

private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ......
        final IActivityManager mgr = ActivityManager.getService();
        try {
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        ......
    } else {
        ......
    }
    ......
}
```

ActivityThread.attach中获取到ActivityManager代理对象，然后调用其attachApplication方法，参数是mAppThread，其类型为ApplicationThread。

AMS收到应用进程attachApplication请求后，将调度应用进程ApplicationThread.bindApplication执行。

```
>>> com.android.server.am.ActivityManagerService


private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {

    ......
    ProcessRecord app;
    ......
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
        ......
    } else {
        app = null;
    }

    ......

    app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
    mOomAdjuster.setAttachingSchedGroupLocked(app);
    app.forcingToImportant = null;
    updateProcessForegroundLocked(app, false, 0, false);
    app.hasShownUi = false;
    app.setDebugging(false);
    app.setCached(false);
    app.killedByAm = false;
    app.killed = false;

    ......

    try {
    ......
    thread.bindApplication(processName, appInfo, providerList, null, profilerInfo,
            null, null, null, testMode,
            mBinderTransactionTrackingEnabled, enableTrackAllocation,
            isRestrictedBackupMode || !normalMode, app.isPersistent(),
            new Configuration(app.getWindowProcessController().getConfiguration()),
            app.compat, getCommonServicesLocked(app.isolated),
            mCoreSettingsObserver.getCoreSettingsLocked(),
            buildSerial, autofillOptions, contentCaptureOptions,
            app.mDisabledCompatChanges);
        ......
    } catch (Exception e) {
        ......
    }
    return true
}
```

回到到应用进程，ApplicationThread.bindApplication中获取到应用数据后，发送BIND\_APPLICATION消息到主线程消息队列中执行。

```
>>> android.app.ActivityThread

private class ApplicationThread extends IApplicationThread.Stub {
    ......

    @Override
    public final void bindApplication(String processName, ApplicationInfo appInfo,
            ProviderInfoList providerList, ComponentName instrumentationName,
            ProfilerInfo profilerInfo, Bundle instrumentationArgs,
            IInstrumentationWatcher instrumentationWatcher,
            IUiAutomationConnection instrumentationUiConnection, int debugMode,
            boolean enableBinderTracking, boolean trackAllocation,
            boolean isRestrictedBackupMode, boolean persistent, Configuration config,
            CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
            String buildSerial, AutofillOptions autofillOptions,
            ContentCaptureOptions contentCaptureOptions, long[] disabledCompatChanges) {
        ......

        AppBindData data = new AppBindData();
        data.processName = processName;
        data.appInfo = appInfo;
        data.providers = providerList.getList();
        data.instrumentationName = instrumentationName;
        data.instrumentationArgs = instrumentationArgs;
        data.instrumentationWatcher = instrumentationWatcher;
        data.instrumentationUiAutomationConnection = instrumentationUiConnection;
        data.debugMode = debugMode;
        data.enableBinderTracking = enableBinderTracking;
        data.trackAllocation = trackAllocation;
        data.restrictedBackupMode = isRestrictedBackupMode;
        data.persistent = persistent;
        data.config = config;
        data.compatInfo = compatInfo;
        data.initProfilerInfo = profilerInfo;
        data.buildSerial = buildSerial;
        data.autofillOptions = autofillOptions;
        data.contentCaptureOptions = contentCaptureOptions;
        data.disabledCompatChanges = disabledCompatChanges;
        sendMessage(H.BIND_APPLICATION, data);
    }
}
```

在主线程消息Handler处理中，收到BIND\_APPLICATION后，调用handleBindApplication方法。

```
class H extends Handler {
    ......
    public void handleMessage(Message msg) {
        
        switch (msg.what) {
            case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            AppBindData data = (AppBindData)msg.obj;
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
        ......
    }
}
```

在handleBindApplication中将先后创建ContextImpl，Instrumentation，Application，并调用Application的

```
private void handleBindApplication(AppBindData data) {
    //step 1: 创建LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    //step 2: 创建ContextImpl对象;
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);

    //step 3: 创建Instrumentation
    mInstrumentation = new Instrumentation();

    //step 4: 创建Application对象; 执行attachBaseContext回调
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;

    //step 5: 安装providers
    List<ProviderInfo> providers = data.providers;
    installContentProviders(app, providers);

    //step 6: 执行Application.Create回调
    mInstrumentation.callApplicationOnCreate(app);
}
```

在创建Application后，执行Application.onCreate之前，即Application.attachBaseContext与onCreate之间，调用installContentProviders执行ContentProvider初始化。

```
public Application makeApplication(boolean forceDefaultAppClass,
                Instrumentation instrumentation) {
    ......
    Application app = null;
    ......

    try {
        final java.lang.ClassLoader cl = getClassLoader();
        ......
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) { 
        ......
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    return app;
}
```

在Instrumentation中通过classloader创建Application实例。

```
>>> android.app.Instrumentation

public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}

public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
    @NonNull String className)
    throws InstantiationException, IllegalAccessException, ClassNotFoundException {
return (Application) cl.loadClass(className).newInstance();
}
```

创建Application后，调用attachBaseContext方法。

```
>>> android.app.Application

final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}
```

## 3. Context核心方法

| 对象             | 方法                    | 返回值类型           | 含义                            |
| -------------- | --------------------- | --------------- | ----------------------------- |
| Activity       | getApplication()      | Application     | 获取Activity所属的mApplication     |
| Service        | getApplication()      | Application     | 获取Service所属的mApplication      |
| ContextWrapper | getBaseContext        | ContextImpl     | 获取mBase,即ContextImpl          |
| ContextWrapper | getApplicationContext | Application     | 见解释                           |
| ContextImpl    | getApplicationContext | Application     | 见解释                           |
| ContextImpl    | getOuterContext       | ContextImpl     | 获取mOuterContext               |
| ContextImpl    | getApplicationInfo    | ApplicationInfo | mPackageInfo.mApplicationInfo |

```
class ContextImpl extends Context {
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
}

//上述mPackageInfo的数据类型为LoadedApk
public final class LoadedApk {
    Application getApplication() {
        return mApplication;
    }
}

//上述mMainThread为ActivityThread
public final class ActivityThread {
    public Application getApplication() {
        return mInitialApplication;
    }
}
```

## 4. 参考

* [Android Context的设计思想和源码分析](https://duanqz.github.io/2017-12-25-Android-Context)
* [理解Android Context](http://gityuan.com/2017/04/09/android_context/)

