# Android基础-APP启动全解析

![](<../../.gitbook/assets/image (57).png>)

## 1. 启动流程

1\. Launcher 捕获点击事件，其过程为 `Launcher#onClick` -> `Launcher#onClickAppShortcut` -> `Launcher#startAppShortcutOrInfoActivity` -> `Launcher#startActivitySafely` -> `Activity#startActivity`，其 Launcher3 相关源码如下所示：

```
>>> launcher3/Launcher.java

public void onClick(View v) {
    ...
    Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        onClickAppShortcut(v);
    }
    ...
}

protected void onClickAppShortcut(final View v) {
    ...
    // Start activities
    startAppShortcutOrInfoActivity(v);
}

private void startAppShortcutOrInfoActivity(View v) {
    ItemInfo item = (ItemInfo) v.getTag();
    Intent intent;
    // 应用程序安装的时候根据 AndroidManifest.xml 由 PackageManagerService 解析并保存的
    if (item instanceof PromiseAppInfo) {
        PromiseAppInfo promiseAppInfo = (PromiseAppInfo) item;
        intent = promiseAppInfo.getMarketIntent();
    } else {
        intent = item.getIntent();
    }
    ...
    boolean success = startActivitySafely(v, intent, item);
    ...
}

public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    ...
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    ...
    startActivity(intent, optsBundle);
    ...
}
```

2\. 以 API 29源码为例， `Acitvity#startActivity`，调用的是 `Activity#startActivityForResult`，其中调用到了 `Instrumentation#execStartActivity` 这个方法，源码如下所示：

```
>>> android.app.Activity

public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}


public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
    .......
}

>>> android.app.Instrumentation

public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    ......
    try {
        ......
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        ......
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

3\. 在 `Instrumentation#execStartActivity` 中我们可以发现它调用了 `ActivityManager#getService()#startActivity`，其 `ActivityManager#getService()` 是采用单例，返回的是实现 `IActivityManager` 类型的 `Binder` 对象，它的具体实现是在 `ActivityManagerService` 中。

```
>>> android.app.ActivityTaskManager

public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}


private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
```

4\. 我们再到 `ActivityManagerService#startActivity` 查看其源码，发现其调用了 `ActivityManagerService#startActivityAsUser`，该方法又调用了 `ActivityStarter#startActivityMayWait`，源码如下所示：

```
>>> com.android.server.am.ActivityManagerService

public int startActivity(IApplicationThread caller, String callingPackage,
    Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
    int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
return mActivityTaskManager.startActivity(caller, callingPackage, intent, resolvedType,
        resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions);
}


>>> com.android.server.wm.ActivityTaskManagerService

public final int startActivity(IApplicationThread caller, String callingPackage,
    Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
    int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
        resultWho, requestCode, startFlags, profilerInfo, bOptions,
        UserHandle.getCallingUserId());
}

public int startActivityAsUser(IApplicationThread caller, String callingPackage,
    Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
    int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
        resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
        true /*validateIncomingUser*/);
}

int startActivityAsUser(IApplicationThread caller, String callingPackage,
    Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
    int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
    boolean validateIncomingUser) {
enforceNotIsolatedCaller("startActivityAsUser");

userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
        Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

// TODO: Switch to user app stacks here.
return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
        .setCaller(caller)
        .setCallingPackage(callingPackage)
        .setResolvedType(resolvedType)
        .setResultTo(resultTo)
        .setResultWho(resultWho)
        .setRequestCode(requestCode)
        .setStartFlags(startFlags)
        .setProfilerInfo(profilerInfo)
        .setActivityOptions(bOptions)
        .setMayWait(userId)
        .execute();
}
```

5\. 可以看到启动Activity的工作交给`getActivityStartController().obtainStarter` , 进入到`ActivityStarter.execute()中执行`

```
>>> com.android.server.wm.ActivityStarter

int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                    mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        }
    } finally {
        onExecutionComplete();
    }
}

private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
    // Refuse possible leaked file descriptors
        ......

        final ActivityRecord[] outRecord = new ActivityRecord[1];
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);
        .......
}


private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    int result = START_CANCELED;
    final ActivityStack startedActivityStack;
    try {
        ......
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
    } finally {
        ......
    }

    postStartActivityProcessing(r, result, startedActivityStack);

    return result;
}

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        ......
        mRootActivityContainer.resumeFocusedStacksTopActivities();
        ......
}


>>> com.android.server.wm.RootActivityContainer

boolean resumeFocusedStacksTopActivities() {
    return resumeFocusedStacksTopActivities(null, null, null);
}

boolean resumeFocusedStacksTopActivities(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

    ......
    result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    ......
    return result;
}
```

5\. 流程进入到中 `ActivityStack`继续执行

```
>>> com.android.server.wm.ActivityStack

boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    boolean result = false;
    try {
        ......
        result = resumeTopActivityInnerLocked(prev, options);
        ......
    } finally {
        mInResumeTopActivity = false;
    }

    return result;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ......
    mStackSupervisor.startSpecificActivityLocked(next, true, false);
    ......
}


>>> com.android.server.wm.ActivityStackSupervisor

void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    ......
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }
    ......
}


boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {
    ......
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    ......
}
```

6\. 至此，启动Activity过程将进入到Client ActivityThread中, 安排事务到APP进程执行

```
>>> android.app.ActivityThread

private class ApplicationThread extends IApplicationThread.Stub {
    @Override
    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
    }
}

public void handleMessage(Message msg) {
    switch (msg.what) {
    case EXECUTE_TRANSACTION:
        final ClientTransaction transaction = (ClientTransaction) msg.obj;
        mTransactionExecutor.execute(transaction);
        ......    
    }
}


public void execute(ClientTransaction transaction) {
    ......
    executeLifecycleState(transaction);
    ......
}
```

回到APP进程, 在ClientTransaction的schedule()中调用ActivityThread内部类ApplicationThread的scheduleTransaction()：

```
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

在ApplicationThread的scheduleTransaction()调用了ActivityThread也就是其父类ClientTransactionHandler的scheduleTransaction()：

```
@Override
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

在ClientTransactionHandler的scheduleTransaction()，通过H发送了EXECUTE\_TRANSACTION消息：

```
/** Prepare and schedule transaction for execution. */
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

在H中接收EXECUTE\_TRANSACTION消息，并调用了TransactionExecutor的execute()：

```
case EXECUTE_TRANSACTION:
    final ClientTransaction transaction = (ClientTransaction) msg.obj;
    mTransactionExecutor.execute(transaction);
    if (isSystem()) {
        // Client transactions inside system process are recycled on the client side
        // instead of ClientLifecycleManager to avoid being cleared before this
        // message is handled.
        transaction.recycle();
    }
    // TODO(lifecycler): Recycle locally scheduled transactions.
    break;
```

如果对应的是onPause()流程，将会在TransactionExecutor的execute()调用executeLifecycleState()，并继续调用ActivityLifecycleItem也就是子类PauseActivityItem的execute()：

```
lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
```

在PauseActivityItem的execute()调用ClientTransactionHandler也就是子类ActivityThread的handlePauseActivity()：

```
@Override
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
    client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
            "PAUSE_ACTIVITY_ITEM");
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

而后将继续在ActivityThread的performPauseActivityIfNeeded()调用Instrumentation的callActivityOnPause()：

```
mInstrumentation.callActivityOnPause(r.activity);
```

而后调用Activity的performPause()：

```
public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

最后调用Activity的onPause()：

```
final void performPause() {
    dispatchActivityPrePaused();
    mDoReportFullyDrawn = false;
    mFragments.dispatchPause();
    mCalled = false;
    onPause();
    writeEventLog(LOG_AM_ON_PAUSE_CALLED, "performPause");
    mResumed = false;
    if (!mCalled && getApplicationInfo().targetSdkVersion
            >= android.os.Build.VERSION_CODES.GINGERBREAD) {
        throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPause()");
    }
    dispatchActivityPostPaused();
}
```

如果对应的是onCreate()流程，将会在TransactionExecutor的execute()调用executeCallbacks()，并继续调用ClientTransactionItem 也就是子类LaunchActivityItem的execute()：

```
item.execute(mTransactionHandler, token, mPendingActions);
```

在LaunchActivityItem的execute()调用ClientTransactionHandler也就是子类ActivityThread的handleLaunchActivity()：

```
@Override
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

而后将继续在ActivityThread的performLaunchActivity()调用Instrumentation的newActivity()使用反射来创建Activity实例：

```
java.lang.ClassLoader cl = appContext.getClassLoader();
activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
public Activity newActivity(ClassLoader cl, String className,
        Intent intent)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    String pkg = intent != null && intent.getComponent() != null
            ? intent.getComponent().getPackageName() : null;
    return getFactory(pkg).instantiateActivity(cl, className, intent);
}
public @NonNull Activity instantiateActivity(@NonNull ClassLoader cl, @NonNull String className,
        @Nullable Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Activity) cl.loadClass(className).newInstance();
}
```

而后继续在ActivityThread的performLaunchActivity()调用Activity的attach()：

```
activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback,
        r.assistToken);
```

创建好Activity实例后，将会调用Instrumentation的callActivityOnCreate()：

```
if (r.isPersistable()) {
    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
} else {
    mInstrumentation.callActivityOnCreate(activity, r.state);
}
```

而后调用Activity的performCreate()：

```
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

最后调用Activity的onCreate()：

```
@UnsupportedAppUsage
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    dispatchActivityPreCreated(icicle);
    mCanEnterPictureInPicture = true;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
    writeEventLog(LOG_AM_ON_CREATE_CALLED, "performCreate");
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    dispatchActivityPostCreated(icicle);
}
```

## 2. 进程创建过程

ActivityStackSupervisor.startSpecificActivityLocked, 这个方法里会去判断ProcessRecord 是否是空, 如果是空会去初始化这个进程, 并且创建一个ActivityThread 在这里就为一个新的程序创建一个主线程,如果当前进程存在会执行 realStartActivityLocked 方法去打对应的Activity页面.

```
>>> com.android.server.wm.ActivityStackSupervisor

void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

 
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            ......
        }
    }

    ......

    try {
        ......
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        final Message msg = PooledLambda.obtainMessage(
                ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
        mService.mH.sendMessage(msg);
    } finally {
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

进程启动过程参考: [http://gityuan.com/2016/03/26/app-process-create/](http://gityuan.com/2016/03/26/app-process-create/)\
\
参考: [https://juejin.cn/post/6844903968997376013](https://juejin.cn/post/6844903968997376013)
