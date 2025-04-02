# Android基础之渲染机制全解析

![](<../../.gitbook/assets/image (53).png>)

## 1. **基础概念**

### **1.1 核心组成**

![](<../../.gitbook/assets/image (76).png>)

上图列举了Android应用进程中的几个核心类及他们之间的关系，下面具体来认识一下：

### 1.2 Window

#### **Window的概念**

Window 表示一个窗口的概念，是一个抽象的概念（并不是因为它是个接口，它也有对应的实现类，如PhoneWindow），每个Window对应着一个根View和一个ViewRootImpl，这两者由WindowManagerGlobal持有管理，WindowManagerGlobal控制ViewRootImpl通过WindowSession与WMS通讯，通知根View在界面上的add、update、remove，这也可以看作是Window的add、update、remove。一个Window必定仅包含一个根View，也可以说Window以根View的形式存在。

* Window和View通过ViewRootImpl建立联系
* Window并不是实际存在的，而是以View的形式存在

#### Window的类型

* **应用Window：**&#x5BF9;应于一个Activity。加载Activity由ams完成，创建一个应用窗口只能在Activity内部完成（层级1\~99）
* **子Window：**&#x5B50;Window不能单独存在，必须依附在父Window上，子Window有比如PopWindow、Dialog。
* **系统Window：**&#x7CFB;统Window需要申明权限，一般浮在屏幕最上面（其z-ordered大于另外两种Window），如Toast

### 1.3 ViewRootImpl

ViewRootImpl是应用进程运转的发动机，可以看到ViewRootImpl内部包含mView、mSurface、Choregrapher，mView代表整个控件树，mSurfacce代表画布，应用的UI渲染会直接放到mSurface中，Choregorapher使得应用请求vsync信号，接收信号后开始渲染流程。

### 1.4 Surface

Surface在源码中的解释为“_Handle onto a raw buffer that is being managed by the screen compositor_“，翻译成中文就是“由屏幕显示内容合成器(screen compositor)所管理的原始缓冲区的句柄”，这句话包括下面两个意思：

1. &#x20;通过Surface（因为Surface是句柄）就可以获得原生缓冲器以及其中的内容。就像在C++语言中，可以通过一个文件的句柄，就可以获得文件的内容一样。
2. 原始缓冲区（a raw buffer）是用于保存当前窗口的像素数据的。

引伸地，可以认为Android中的Surface就是一个用来画图形（graphics）或图像（image）的地方。



## 2. ViewRootImpl详解



## 3. 渲染过程

![](<../../.gitbook/assets/image (350).png>)

上图的应用启动过程前面的总结已经有很详细的描述，我们从Activity的创建开始到界面的渲染显示，完成跟踪一遍这个过程。

### **2.1 Activity#attach**

```
>>> android.app.ActivityThread

 => handleLaunchActivity()
 ==> performLaunchActivity()


private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ......
    // 1.反射创建Activity
    java.lang.ClassLoader cl = appContext.getClassLoader();
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    
    // 2. 调用其attach方法
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback,
        r.assistToken);
    ......
    return activity;
}
```

Activity创建以后立即回调其attach方法

```
>>> android.app.Activity

final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {

    ......

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(mWindowControllerCallback);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ......

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    ......
}
```

在Activity#Attach中，创建PhoneWindow对象，PhoneWindow是Window的唯一实现类。

```
>>> com.android.internal.policy.PhoneWindow

public PhoneWindow(Context context) {
    super(context);
    mLayoutInflater = LayoutInflater.from(context);
    mRenderShadowsInCompositor = Settings.Global.getInt(context.getContentResolver(),
            DEVELOPMENT_RENDER_SHADOWS_IN_COMPOSITOR, 1) != 0;
}
```

创建PhoneWindow对象后，设置其WindowManager。

```
>>> android.view.Window

public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

创建WindowManagerImpl

```
>>> android.view.WindowManagerImpl

public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

    ......
    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }
    
    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
    ......
}
```

WindowManagerImpl中有一成员变量WindowManagerGlobal，通过WindowManagerGlobal.getInstance获取实例。

```
>>> android.view.WindowManagerGlobal

public static WindowManagerGlobal getInstance() {
    synchronized (WindowManagerGlobal.class) {
        if (sDefaultWindowManager == null) {
            sDefaultWindowManager = new WindowManagerGlobal();
        }
        return sDefaultWindowManager;
    }
}
```

上述步骤创建的Window、WindowManager、WindowManagerImpl、WindowManagerGlobal对象关系如下所示：

![](<../../.gitbook/assets/image (237).png>)

### **2.2 Activity#onCreate**

我们应用程序定义Activity第一个回调即onCreate，一般会执行setContentView设置我们的布局。

```
void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}
```

setContentView会调用getWindow().setContentView()

```
android.app.Activity

public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

前面提到Window的唯一实现类是PhoneWindow，所以这里进入到PhoneWindow的setContentView执行。

```
>>> com.android.internal.policy.PhoneWindow

public void setContentView(int layoutResID) {

    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

setContentView中，首先判断mContentParent是否为空，mContentParent是我们布局文件的父布局，对应R.id.content。mContentParent为空，先执行installDecor()。

```
>>> com.android.internal.policy.PhoneWindow

private DecorView mDecor;

private void installDecor() {
    ......
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        ......
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        ......
    }
}
```

installDecor中通过generateDecor创建DecorView。

```
>>> com.android.internal.policy.PhoneWindow

protected DecorView generateDecor(int featureId) {
    ......
    return new DecorView(context, featureId, this, getAttributes());
}
```

直接new DecorView，DecorView继承于FrameLayout。

```
>>> com.android.internal.policy.DecorView

public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {

}
```

创建好DecorView以后，继续调用generateLayout(mDecor)生成mContentParent。

```
>>> com.android.internal.policy.PhoneWindow

protected ViewGroup generateLayout(DecorView decor) {
    ......
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ......
    return contentParent;
}
```

contentParent直接对应ID\_ANDROID\_CONTENT即R.id.content，即应用自定义布局的父容器。继续回到setContentView函数中，在完成DecorView与ContentParent的创建后，调用mLayoutInflater.inflate将我们应用布局的layoutResID加载到mContentParent中。

```
>>> com.android.internal.policy.PhoneWindow

public void setContentView(int layoutResID) {

    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    ......
    mLayoutInflater.inflate(layoutResID, mContentParent);
    ......
}
```

上述步骤创建的对象关系如下图所示：

![](<../../.gitbook/assets/image (13).png>)

![](<../../.gitbook/assets/image (343).png>)

### 2.3 Activity#onResume

在前面的Activity#onAttach，Activity#onCreate过程中，创建了Window，及以DecorView为根结点的树形结构，但是View并没有被添加到Window中。

```
>>> android.app.ActivityThread

public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
    ......
    // 回调Activity.onResume
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);

    final Activity a = r.activity;

    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r . window . getDecorView ();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a . getWindowManager ();
        WindowManager.LayoutParams l = r . window . getAttributes ();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;

        ......
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                // 将decorView添加到Window
                wm.addView(decor, l);
            } else {
                ......
            }
        }
    }
    ......
}
```

Activity#onResume中调用wm.addView将Decor添加到window中。由Activity#attach中的流程可知，这里是调用的WindowManagerImpl.addView。

```
>>> android.view.WindowManagerImpl

public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
            mContext.getUserId());
}
```

进一步调用到WindowManagerGlobal.addView

```
>>> android.view.WindowManagerGlobal

public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow, int userId) {

    ......
    ViewRootImpl root;
    View panelParentView = null;
    ......
    synchronized (mLock) {
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView, userId);
        } catch (RuntimeException e) {
           ......
        }
    }
}
```

WindowManagerGlobal.addView中是比较关键的一步，创建ViewRootImpl对象并且调用其setView方法。

```
```



### 2.4 Surface的创建

![](<../../.gitbook/assets/image (308).png>)

ViewRootImpl在构造的时候就new了一个surface。但其实这个新new的Surface并没有什么逻辑，它的构造函数是空的。那么Surface到底是在哪里创建的呢？

```
>>> android.view.ViewRootImpl

public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    ......
    public final Surface mSurface = new Surface();
    ......    
}
```

我们还是回到ViewRootImpl.setView()来看一下:

```
>>> android.view.ViewRootImpl

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    requestLayout(); //susion 请求layout。先添加到待渲染队列中  
    ...
    //WindowManagerService会创建mWindow对应的WindowState
    res = mWindowSession.addToDisplay(mWindow, ...); 
}
```

即在向WindowManagerService请求创建WindowState之前，调用了requestLayout(),这个方法会引起ViewRootImpl所管理的整个view tree的重新渲染。它最终会调用到scheduleTraversals():

```
>>> android.view.ViewRootImpl

void scheduleTraversals() {
    ...
    mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    ...
}
```

scheduleTraversals()会通过Choreographer来post一个mTraversalRunnable，Choreographer接收显示系统的时间脉冲(垂直同步信号-VSync信号，16ms发出一次)，在下一个frame渲染时控制执行这个mTraversalRunnable。

但是mTraversalRunnable的执行至少要在应用程序与SurfaceFlinger建立连接之后。这是因为渲染操作是由SurfaceFlinger负责调度了，如果应用程序还没有与SurfaceFlinger创建连接，那SurfaceFlinger当然不会渲染这个应用程序。所以在执行完mWindowSession.addToDisplay(mWindow, ...)之后，才会执行mTraversalRunnable:

```
>>> android.view.ViewRootImpl

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

doTraversal()会调用到ViewRootImpl.performTraversals()，后面的流程就是一个view tree的measure/layout/draw的控制方法:

```
>>> android.view.ViewRootImpl

private void performTraversals() {
    finalView host = mView; //mView是一个Window的根View，对于Activity来说就是DecorView
    ...
    relayoutWindow(params, viewVisibility, insetsPending);
    ...
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...         
    performLayout(lp, mWidth, mHeight);
    ...
    performDraw();
    ...
}
```

直到走到这里，Surface才开始被创建，它的具体创建是由relayoutWindow(params, viewVisibility, insetsPending)这个方法来完成的。

```
>>> android.view.ViewRootImpl

private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
        boolean insetsPending) throws RemoteException {
    ......
    int relayoutResult = mWindowSession.relayout(mWindow, ......,
                       mSurfaceControl, ......);
    ......  
    mSurface.copyFrom(mSurfaceControl);
    ......
    return relayoutResult;
}
```

上面省略了mWindowSession.relayout()方法的很多参数，不过有一个十分重要的参数我没有省略，就是mSurfaceControl。前面已经分析了它就是一个空的Surface对象。所以，真正的Surface创建是由SurfaceControl完成的。

先来看一下relayoutWindow的执行过程，mWindowSession.relayout()会调用到WindowManagerService.relayoutWindow()

```
>>> com.android.server.wm.WindowManagerService

public int relayoutWindow(Session session, IWindow client, ...
        SurfaceControl outSurfaceControl ...) {
    result = createSurfaceControl(outSurfaceControl, outBLASTSurfaceControl,
            result, win, winAnimator);        
}

private int createSurfaceControl(Surface outSurface, int result, WindowState win,WindowStateAnimator winAnimator) {
    ...
    surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
    ...
    surfaceController.getSurface(outSurface);
}
```

winAnimator.createSurfaceLocked其实是通过SurfaceControl的构造函数创建了一个SurfaceControl对象,这个对象的作用其实就是负责维护Surface,Surface其实也是由这个对象负责创建的，我们看一下这个对象的构造方法:

```
>>> android.view.SurfaceControl

long mNativeObject; //成员指针变量，指向native创建的SurfaceControl

private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
            SurfaceControl parent, int windowType, int ownerUid){
    ...
    mNativeObject = nativeCreate(session, name, w, h, format, flags,
        parent != null ? parent.mNativeObject : 0, windowType, ownerUid);
    ...
}
```

即调用了nativeCreate()并返回一个SurfaceControl指针， [android\_view\_SurfaceControl.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_SurfaceControl.cpp;bpv=1;bpt=1;l=227?hl=zh-cn\&gsn=nativeCreate\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_view_SurfaceControl.cpp%23q7lNoVfyDgz6575euonoRu6niIwIwXGTNfjY4-PTgnE)

```
>>> frameworks/base/core/jni/android_view_SurfaceControl.cpp

static jlong nativeCreate(JNIEnv* env, ...) {
    //这个client其实就是前面创建的SurfaceComposerClinent
    sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj)); 
    sp<SurfaceControl> surface; //创建成功之后，这个指针会指向新创建的SurfaceControl
    status_t err = client->createSurfaceChecked(String8(name.c_str()), w, h, format, &surface, flags, parent, windowType, ownerUid);
    ...
    return reinterpret_cast<jlong>(surface.get()); //返回这个SurfaceControl的地址
}
```

即调用到SurfaceComposerClient.createSurfaceChecked()，[SurfaceComposerClient.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/SurfaceComposerClient.cpp;drc=master;bpv=1;bpt=1;l=1638?hl=zh-cn\&gsn=createSurfaceChecked\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Flibs%2Fgui%2FSurfaceComposerClient.cpp%23HFAi_R3VaoeXfTuj-gJjKy7agkyT2gR4xOfcS-DlPrg\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Flibs%2Fgui%2Finclude%2Fgui%2FSurfaceComposerClient.h%23qt92rfUpZN7uTyBj4Wcv_BkdcADqNKAw3g8VNFv3X74)

```
>>> frameworks/native/libs/gui/SurfaceComposerClient.cpp

//outSurface会指向新创建的SurfaceControl
status_t SurfaceComposerClient::createSurfaceChecked(...sp<SurfaceControl>* outSurface..) 
{
    sp<IGraphicBufferProducer> gbp; //这个对象很重要
    ...
    err = mClient->createSurface(name, w, h, format, flags, parentHandle, windowType, ownerUid, &handle, &gbp);
    if (err == NO_ERROR) {
        //SurfaceControl创建成功, 指针赋值
        *outSurface = new SurfaceControl(this, handle, gbp, true);
    }
    return err;
}
```

上面这个方法实际上是调用Client.createSurface()来创建一个Surface。在创建时有一个很重要的参数**sp\<IGraphicBufferProducer> gbp**，在下面源码分析中我们也要重点注意它。这是因为应用所渲染的每一帧，实际上都会添加到IGraphicBufferProducer中，来等待SurfaceFlinger的渲染。

[Client.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/services/surfaceflinger/Client.cpp;bpv=1;bpt=1;l=79?q=client.cpp\&ss=android%2Fplatform%2Fsuperproject\&hl=zh-cn\&gsn=createSurface\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Fservices%2Fsurfaceflinger%2FClient.cpp%234InIFQNL-5oAHt9Ave-X4lRwnAB6z6UklA8XRYpK-ig\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Fservices%2Fsurfaceflinger%2FClient.h%235tqjgn8giyIK9Mf10o_8jlIYTUi4WprSscX_qB5_2fs)

```
>>> frameworks/native/services/surfaceflinger/Client.cpp

status_t Client::createSurface(const String8& name, uint32_t w, uint32_t h, PixelFormat format,
                               uint32_t flags, const sp<IBinder>& parentHandle,
                               LayerMetadata metadata, sp<IBinder>* handle,
                               sp<IGraphicBufferProducer>* gbp, uint32_t* outTransformHint) {
    // gbp 直接透传到了SurfaceFlinger
    return mFlinger->createLayer(name, this, w, h, format, flags, std::move(metadata), handle, gbp,
                                 parentHandle, nullptr, outTransformHint);
}
```

不是说好的要创建Surface呢？怎么变成mFlinger->createLayer()？其实，**Surface在SurfaceFlinger中对应的实体其实是Layer。**&#x7EE7;续看一下mFlinger->createLayer()。

[SurfaceFlinger.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp;drc=master;bpv=1;bpt=1;l=3827?hl=zh-cn\&gsn=createLayer\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Fservices%2Fsurfaceflinger%2FSurfaceFlinger.cpp%23voLYNdurOdlqGMdtocWY5C4t3S1bjuXgRfTYqCEMxvQ\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Fservices%2Fsurfaceflinger%2FSurfaceFlinger.h%23XU13znDkxX7At-HjuQwUybR0sIOqk1OfqDWkSamQG50)

```
>>> frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

status_t SurfaceFlinger::createLayer(const String8& name,const sp<Client>& client...)
{
    status_t result = NO_ERROR;
    sp<Layer> layer; //将要创建的layer
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceNormal:
            result = createBufferLayer(client,
                    uniqueName, w, h, flags, format,
                    handle, gbp, &layer); // 注意gbp，这时候还没有构造呢！
            break;
            ... //Layer 分为好几种，这里不全部列出
    }
    ...
    //这个layer和client相关联, 添加到Client的mLayers集合中。
    result = addClientLayer(client, *handle, *gbp, layer, *parent);  
    ...
    return result;
}
```

从SurfaceFlinger.createLayer()方法可以看出Layer分为好几种。我们这里只对普通的BufferLayer的创建做一下分析，看createBufferLayer()。

```
status_t SurfaceFlinger::createBufferLayer(const sp<Client>& client... sp<Layer>* outLayer)
{
    ...
    sp<BufferLayer> layer = new BufferLayer(this, client, name, w, h, flags);
    status_t err = layer->setBuffers(w, h, format, flags);  //设置layer的宽高
    if (err == NO_ERROR) {
        *handle = layer->getHandle(); //创建handle
        *gbp = layer->getProducer(); //创建 gbp IGraphicBufferProducer
        *outLayer = layer; //把新建的layer的指针拷贝给outLayer,这样outLayer就指向了新建的BufferLayer
    }
    return err;
}
```

前面说过IGraphicBufferProducer(gbp)是一个很重要的对象，它涉及到SurfaceFlinger的渲染逻辑，下面我们就看一下这个对象的创建逻辑。

```
sp<IGraphicBufferProducer> BufferLayer::getProducer() const {
    return mProducer;
}
```

即mProducer其实是Layer的成员变量，它的创建时机是Layer第一次被使用时:

```
void BufferLayer::onFirstRef() {
    ...
    BufferQueue::createBufferQueue(&producer, &consumer, true); 
    mProducer = new MonitoredProducer(producer, mFlinger, this);
    ...
}
```

所以mProducer的实例是MonitoredProducer,但其实它只是一个装饰类，它实际功能都委托给构造它的参数producer:

BufferQueue.cpp

```
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
    ...
    sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core, consumerIsSurfaceFlinger));
    //注意这个consumer
    ...
    sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core)); 
    *outProducer = producer;
    *outConsumer = consumer;
}
```

所以实际实现mProducer的工作的queueProducer是BufferQueueProducer。

**所以构造一个SurfaceControl所做的工作就是创建了一个SurfaceControl，并让SurfaceFlinger创建了一个对应的Layer，Layer中有一个IGraphicBufferProducer，它的实例是BufferQueueProducer。**

![](<../../.gitbook/assets/image (26).png>)

了解了上面这个过程，继续回到WindowManagerService.createSurfaceControl()中，来看一下java层的`Surface`对象到底是什么。

```
>>> com.android.server.wm.WindowManagerService

public int relayoutWindow(Session session, IWindow client, ...
        SurfaceControl outSurfaceControl ...) {
    result = createSurfaceControl(outSurfaceControl, outBLASTSurfaceControl,
            result, win, winAnimator);        
}

private int createSurfaceControl(Surface outSurface, int result, WindowState win,WindowStateAnimator winAnimator) {
    ...
    surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
    ...
    surfaceController.getSurface(outSurface);
}
```

上面我们已经了解了winAnimator.createSurfaceLocked的整个过程，我们看一下surfaceController.getSurface(outSurface), surfaceController是WindowSurfaceController的实例:

```
//WindowSurfaceController.java
void getSurface(Surface outSurface) {
    outSurface.copyFrom(mSurfaceControl);
}

//Surface.java
public void copyFrom(SurfaceControl other) {
    ...
    long surfaceControlPtr = other.mNativeObject;
    ...
    long newNativeObject = nativeGetFromSurfaceControl(surfaceControlPtr);
    ...
    mNativeObject = ptr; // mNativeObject指向native创建的Surface
}
```

即`Surface.copyFrom()`方法调用`nativeGetFromSurfaceControl()`来获取一个指针，这个指针是根据前面创建的`SurfaceControl`的指针来寻找的，即传入的参数`surfaceControlPtr`:

[android\_view\_Surface.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_Surface.cpp;bpv=1;bpt=1;l=282?q=android_view_Surface.cpp%20\&ss=android%2Fplatform%2Fsuperproject\&hl=zh-cn\&gsn=nativeGetFromSurfaceControl\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_view_Surface.cpp%236fHazIllUJNupubgTLo9Hb2C0_-0SPaEh4OYaCECpJg)

```
static jlong nativeGetFromSurfaceControl(JNIEnv* env, jclass clazz, jlong surfaceControlNativeObj) {
    sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj)); //把java指针转化内native指针
    sp<Surface> surface(ctrl->getSurface()); //直接构造一个Surface，指向 ctrl->getSurface()
    if (surface != NULL) {
        surface->incStrong(&sRefBaseOwner); //强引用
    }
    return reinterpret_cast<jlong>(surface.get());
}
```

这里的ctrl指向前面创建的SurfaceControl，继续追溯ctrl->getSurface():

[SurfaceControl.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/SurfaceControl.cpp;bpv=1;bpt=1;l=122?q=SurfaceControl.cpp%20\&ss=android%2Fplatform%2Fsuperproject\&hl=zh-cn\&gsn=generateSurfaceLocked\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Flibs%2Fgui%2FSurfaceControl.cpp%23A1Pyy-kXgOzcuEW0sxfHu5dbtKNcpNV3Y5Ks4HFAcVU\&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fnative%2Flibs%2Fgui%2Finclude%2Fgui%2FSurfaceControl.h%23K1aecLavL_ZvkNvZ394y4z4DCo0VGaeexGZ7Hr7iYqI)

```
>>> frameworks/native/libs/gui/SurfaceControl.cpp


sp<Surface> SurfaceControl::getSurface() const
{
    Mutex::Autolock _l(mLock);
    if (mSurfaceData == 0) {
        return generateSurfaceLocked();
    }
    return mSurfaceData;
}

sp<Surface> SurfaceControl::generateSurfaceLocked() const
{
    //这个mGraphicBufferProducer其实就是上面分析的BufferQueueProducer
    mSurfaceData = new Surface(mGraphicBufferProducer, false); 
    return mSurfaceData;
}
```

即直接new了一个nativie的Surface返回给java层，java层的Surface指向的就是native层的Surface。

所以Surface的实际创建可以用下图表示:

![](<../../.gitbook/assets/image (85).png>)

ViewRootImpl剖析

上面的框架图提到ViewRootImpl有个非常重要的对象Choreographer，整个应用布局的渲染依赖这个对象发动，应用要求渲染动画或者更新画面布局时都会用到Choreographer，接收vsync信号也依赖于Choreographer。

我们以一个View控件调用invalidate函数来分析应用如何接收vsync、以及如何更新UI的。Activity中的某个控件调用invalidate以后，会逆流到根控件，最终到达调用到ViewRootImpl.java#Invalidate

**invalidate函数**

```
void invalidate() {
  mDirty.set(0, 0, mWidth, mHeight);
  if (!mWillDrawSoon) {
  scheduleTraversals();
  }
}
```

**scheduleTraversals函数**

```
void scheduleTraversals() {
       if (!mTraversalScheduled) {
           mTraversalScheduled = true;
           mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
           mChoreographer.postCallback(
                   Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
           if (!mUnbufferedInputDispatch) {
               scheduleConsumeBatchedInput();
           }
           notifyRendererOfFramePending();
           pokeDrawLockIfNeeded();
       }
   }
```

从上面的代码看到Invalidate最终调用到mChoreographer.postCallback，这代码的含义：应用程序请求vsync信号，收到vsync信号以后会调用mTraversalRunnable，接下来看下应用程序如何通过Choreographer接收vsync信号：

```
//Choreographer.java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            // 应用请求vsync信号以后，vsync信号分发就会回调到这里
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                Log.d(TAG, "Received vsync from secondary display, but we don't support "
                        + "this case yet.  Choreographer needs a way to explicitly request "
                        + "vsync for a specific display to ensure it doesn't lose track "
                        + "of its scheduled vsync.");
                scheduleVsync();
                return;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```

上面onVsync会往消息队列放一个消息，通过下面的FrameHandler进行处理：

```
private final class FrameHandler extends Handler {
       public FrameHandler(Looper looper) {
           super(looper);
       }
       @Override
       public void handleMessage(Message msg) {
           switch (msg.what) {
               case MSG_DO_FRAME:
                   doFrame(System.nanoTime(), 0);
                   break;
               case MSG_DO_SCHEDULE_VSYNC:
                   doScheduleVsync();
                   break;
               case MSG_DO_SCHEDULE_CALLBACK:
                   doScheduleCallback(msg.arg1);
                   break;
           }
       }
   }
```

从systrace中我们经常看到doFrame就是从上面的doFrame打印，这说明应用程序收到了vsync信号要开始渲染布局了，图示如下：

![](<../../.gitbook/assets/image (228).png>)

doFrame函数就开始一次处理input/animation/measure/layout/draw，doFrame代码如下：

```
doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
mFrameInfo.markAnimationsStart();
doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
mFrameInfo.markPerformTraversalsStart();
doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
```

比如上面调用Invalidate的时候已经post了一个CALLBACK\_TRAVERSAL类型的Runnable，这里就会执行到那个Runnable也就是mTraversalRunnable：

```
final class TraversalRunnable implements Runnable {
       @Override
       public void run() {
           doTraversal();
       }
   }
```

**performTraversal函数**

doTraversal内部会调用大名鼎鼎的：performTraversal，这里app就可以进行measure/layout/draw三大流程。需要注意的时候Android在5.1引入了renderthread线程，可以讲draw操作从UIThread解放出来，这样做的好处是，UIThread将绘制指令sync给renderthread以后可以继续执行measure/layout操作，非常有利于提升设备操作体验，如下：

![](<../../.gitbook/assets/image (89).png>)

上面就是应用进程收到vsync信号之后的渲染UI的大概流程，可以看到app进程收到vsync信号以后就开始其measure/layout/draw三大流程，这里面就会回调应用的应用各个空间的onMeasure/onLayout/onDraw，这个部分是在UIThread完成的。UIThread在完成上述步骤以后会绘制指令（DisplayList）同步（sync）给RenderThread，RenderThread会真正的跟GPU通信执行draw动作，systrace图示如下：

![](<../../.gitbook/assets/image (86).png>)

上图中看到doFrame下面会有input/anim(时间短色块比较小）、measure、layout、draw，结合上面的代码分析就清楚了app收到vsync信号的行为，measure/layout/draw的具体分析就涉及到控件系统相关的内容，这块内容本文不作深入分析，提一下draw这个操作，使用硬件加速以后draw部分只是在UIThread中收集绘制命令而已，不做真正的绘制操作.



![](<../../.gitbook/assets/image (376).png>)



## 3. ViewRootImpl详解

ViewRootImpl是View中的最高层级，属于所有View的&#x6839;**`ViewRootImpl不是View，只是实现了ViewParent接口`**），实现了View和WindowManager之间的通信协议，实现的具体细节在WindowManagerGlobal这个类当中。\


![](<../../.gitbook/assets/image (202).png>)





![](<../../.gitbook/assets/image (210).png>)



![](<../../.gitbook/assets/image (63).png>)



既然WMS的作用只是窗口管理，那么图形是怎么绘制的呢？并且这些绘制信息是如何传递给SurfaceFlinger服务的呢？每个View都有自己的onDraw回调，开发者可以在onDraw里绘制自己想要绘制的图像，很明显View的绘制是在APP端，直观上理解，View的绘制也不会交给服务端，不然也太不独立了，可是View绘制的内存是什么时候分配的呢？是谁分配的呢？我们知道每个Activity可以看做是一个图层，其对应一块绘图表面其实就是Surface，Surface绘图表面对应的内存其实是由SurfaceFlinger申请的，并且，内存是APP与SurfaceFlinger间进程共享的。实现机制是基于Linux的共享内存，其实就是MAP+tmpfs文件系统，你可以理解成SF为APP申请一块内存，然后通过binder将这块内存相关的信息传递APP端，APP端往这块内存中绘制内容，绘制完毕，通知SF图层混排，之后，SF再将数据渲染到屏幕。其实这样做也很合理，因为图像内存比较大，普通的binder与socket都无法满足需求，内存共享的示意图如下：

![](<../../.gitbook/assets/image (93).png>)

{% embed url="https://www.jianshu.com/p/e4b19fc36a0e" %}

其实整个Android窗口管理简化的话可以分为以下三部分

* WindowManagerService：WMS控制着Surface画布的添加与次序，动画还有触摸事件
* SurfaceFlinger：SF负责图层的混合，并且将结果传输给硬件显示
* APP端：每个APP负责相应图层的绘制，
* APP与SurfaceFlinger通信：APP与SF图层之间数据的共享是通过匿名内存来实现的。

![](<../../.gitbook/assets/image (357).png>)

![](<../../.gitbook/assets/image (229).png>)



## 4. Choreographer

![](<../../.gitbook/assets/image (24).png>)

## 5. 参考资料

* [https://androidperformance.com/2019/12/01/Android-Systrace-Vsync/](https://androidperformance.com/2019/12/01/Android-Systrace-Vsync/)
* [https://juejin.cn/post/6863756420380196877](https://juejin.cn/post/6863756420380196877)
* [https://mp.weixin.qq.com/s/0OOSmrzSkjG3cSOFxWYWuQ](https://mp.weixin.qq.com/s/0OOSmrzSkjG3cSOFxWYWuQ)
* [http://gityuan.com/2017/02/25/choreographer/](http://gityuan.com/2017/02/25/choreographer/)
*   [https://juejin.cn/post/6844903761292886030](https://juejin.cn/post/6844903761292886030)

