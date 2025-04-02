# Lifecycle



![](<../../.gitbook/assets/image (310).png>)

{% embed url="https://developer.android.com/topic/libraries/architecture/lifecycle#use-cases" %}



### 源码解析

#### 核心成员

![](<../../.gitbook/assets/image (103).png>)

* Lifecycle : 生命周期状态、接口、事件定义
* LifecycleRegistry：实现Lifecycle接口，负责生命周期事件管理与分发
* LifecycleOwner：有生命周期的对象(Activity, Fragment等)，由具体的生命周期对象实现该接口

#### **生命周期事件注册：**



#### 生命周期事件监听与分发：

通过调用Lifecycle.injectIfNeededIn进行生命周期监听，在injectIfNeededIn()内对版本进行了区分：在API 29及以上 直接使用activity的registerActivityLifecycleCallbacks 直接注册了生命周期回调，然后给当前activity添加了ReportFragment，注意这个fragment是空的，没有布局的。

然后， 无论LifecycleCallbacks、还是fragment的生命周期方法 最后都走到了 dispatch(Activity activity, Lifecycle.Event event)方法，其内部使用LifecycleRegistry的handleLifecycleEvent方法处理事件。

而ReportFragment的作用就是获取生命周期而已，因为fragment生命周期是依附Activity的。好处就是把这部分逻辑抽离出来，实现activity的无侵入。

> 利用Fragment对生命周期监测，Glide中也是这样的用法

专门用于分发生命周期事件的Fragment　ReportFragment

```

Fragment public class ReportFragment extends Fragment {
    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            //在API 29及以上，可以直接注册回调 获取生命周期
            activity.registerActivityLifecycleCallbacks(
                    new LifecycleCallbacks());
        }
        //API29以前，使用fragment 获取生命周期
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }
    
    @SuppressWarnings("deprecation")
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        if (activity instanceof LifecycleRegistryOwner) {//这里废弃了，不用看
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }
    
        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);//使用LifecycleRegistry的handleLifecycleEvent方法处理事件
            }
        }
    }
    
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatch(Lifecycle.Event.ON_CREATE);
    }
    @Override
    public void onStart() {
        super.onStart();
        dispatch(Lifecycle.Event.ON_START);
    }
    @Override
    public void onResume() {
        super.onResume();
        dispatch(Lifecycle.Event.ON_RESUME);
    }
    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }
    ...省略onStop、onDestroy
    
    private void dispatch(@NonNull Lifecycle.Event event) {
        if (Build.VERSION.SDK_INT < 29) {
            dispatch(getActivity(), event);
        }
    }
    
    //在API 29及以上，使用的生命周期回调
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
        ...
        @Override
        public void onActivityPostCreated(@NonNull Activity activity,@Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }
        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }
        @Override
        public void onActivityPostResumed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_RESUME);
        }
        @Override
        public void onActivityPrePaused(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_PAUSE);
        }
        ...省略onStop、onDestroy
    }
}
```

#### 生命周期事件处理：

生命周期事件处理全部在 **LifecycleRegistry** 中进行，

```
```



