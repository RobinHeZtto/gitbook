# Android基础之事件分发全解析

当用户触摸屏幕或者按键操作，首次触发的是硬件驱动，驱动收到事件后，将该相应事件写入到输入设备节点， 这便产生了最原生态的内核事件。接着，输入系统取出原生态的事件，经过层层封装后成为KeyEvent或者MotionEvent ；最后，交付给相应的目标窗口(Window)来消费该输入事件。可见，输入系统在整个过程起到承上启下的衔接作用。

Input模块的主要组成：

* Native层的InputReader负责从EventHub取出事件并处理，再交给InputDispatcher；
* Native层的InputDispatcher接收来自InputReader的输入事件，并记录WMS的窗口信息，用于派发事件到合适的窗口；
* Java层的InputManagerService跟WMS交互，WMS记录所有窗口信息，并同步更新到IMS，为InputDispatcher正确派发事件到ViewRootImpl提供保障；

![](<../../.gitbook/assets/image (81).png>)

![](<../../.gitbook/assets/image (367).png>)

{% embed url="https://juejin.cn/post/6865625913309511687" %}

[https://juejin.cn/post/6844903894103883789](https://juejin.cn/post/6844903894103883789)\
\






![](<../../.gitbook/assets/image (335).png>)

![](<../../.gitbook/assets/image (233).png>)

![](<../../.gitbook/assets/image (372).png>)





```
>>> com.android.internal.policy.DecorView

@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

在这个方法中，mWindow就是与Activity关联的PhoneWindow对象，由于DecorView是PhoneWindow创建的，并且通过`setWindow()`方法，DecorView对象持有了PhoneWindow对象的引用。通过`getCallback()`方法，就获得了**Window.Callback**对象，Window.Callback包含了窗口的各种回调接口，Activity就实现了该接口。根据return后的判断，当调用`cb.dispatchTouchEvent(ev)`时，其实调用的就是Activity中的`dispatchTouchEvent()`方法。接下来就是从Activity出发，进一步分析事件分发机制了





```
>>> android.app.Activity;

public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

总结一下上述流程：

* WindowInputEventReceiver收到InputDispatcher分发的事件后，通过enqueueInputEvent方法，将输入事件加入输入事件队列中，并进行处理和转发。
* ViewPostImeInputStage收到输入事件，将事件传递给DecorView的dispatchPointerEvent()方法（实际在是View中定义的方法）
* dispatchPointerEvent()方法通过DecorView中的dispatchTouchEvent()方法，调用了Activity的dispatchTouchEvent()方法，到此事件进入Activity中\




当Activity接受到点击事件后，会传递给Window再传递给DecorView，DecorView就是一个ViewGroup，他的dispatchTouchEvent就会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true，就表示它要拦截事件，事件就会交给这个ViewGroup来处理；如果这个ViewGroup的onInterceptTouchEvent返回false，表示不拦截当前事件，这时，当前事件就会继续传递给它的子元素，子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理。（ViewGroup一般默认不拦截事件）

这里如果子元素是ViewGroup，那么和上面步骤一样，如果子元素是View或者是ViewGroup要自己处理事件。那么要看是否设置了onTouchListener，并且它的onTouch返回值是否为true，如果返回true那么onTouchEvent就不会被回调；如果onTouch返回值为false，那么onTouchEvent就会被调用。

而在onTouchEvent方法中，如果设置了onClickListener，那么它的onClick方法会被调用。

如果一个点击事件产生后，没有View消费这个事件，那么就会交给Activity的onTouchEvent()处理。\


![](<../../.gitbook/assets/image (69).png>)
