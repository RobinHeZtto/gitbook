# Lifecycle



<div align="left"><img src="../../.gitbook/assets/image (499).png" alt=""></div>





![](<../../.gitbook/assets/image (124).png>)



* LifecycleOwner\
  具有Android生命周期的类。 定制的组件可以在内部使用这些生命周期事件，而无需在Activity或Fragment中实现任何代码。（可以理解为被观察的lifecycle对象）
* LifecycleObserver\
  与LifecycleOwner对应，Android生命周期的观察者。（可以理解为lificycle对象的观察者）
* LifecycleEventObserver\
  生命周期事件dispatcher，通过onStateChanged()回调。
*
