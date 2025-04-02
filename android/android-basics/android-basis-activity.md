# Android基础之Activity全解析

Activity的起源，设计初衷，职责边界，



##

## 1. 概念

Android四大应用组件之一，与用户交互的接口，界面的载体。

## 2. 生命周期

![](<../../.gitbook/assets/image (328).png>)

* onCreate()，生命周期第一个方法，表示Activity正在创建过程中。一般做一些初始化工作，setContentView()加载界面布局等。
* onStart()，表示Activity正在启动过程中，Activity可见但不可与用户交互。
* onReStart()，表示Activity正在重新启动过程中，一般Activity由不可见到可见过程中会回调此方法。
* onResume()，Activity显示在前台并可与用户交互。
* onPause()，表示Activity正在停止，可做停止动画，存储数据操作，不能做耗时操作。
* onStop()，表示Activity即将停止，可做轻量级回收工作。
* onDestory()，表示Activity即将销毁，生命周期最后一个方法。

> 从整个生命周期来说，onCreate与onDestory是配对的，标志销毁与创建，整个生命周期只会回调一次。onStart与onStop是配对的，标志Activity是否可见。onResume与onPause是配对的，标志是否处于前台可与用户交互。

> onPause不能做太耗时的操作，否则会影响新的Activity的显示

* 首次启动Activity，系统会先回调onCreate方法然后调用onStart，onResume，进入运行状态。
* 当前Activity被其他Activity覆盖其上或被锁屏（可以理解为没有完全遮挡界面），系统会调用onPause方法，暂停当前Activity的执行。
* 当前Activity由被覆盖状态回到前台或解锁屏，系统会调用onResume方法，再次进入运行状态。
* 当前Activity转到新的Activity界面或按Home键回到主屏，自身退居后台，系统会先调用onPause方法，然后调用onStop方法，进入停滞状态。（如果新Activity采用透明主题，当前Activity不会回调onStop方法）
* 用户后退回到此Activity，系统会先调用onRestart方法，然后调用onStart方法，最后调用onResume方法，再次进入运行状态。
* 用户退出当前Activity，系统先调用onPause方法，然后调用onStop方法，最后调用onDestory方法，结束当前Activity。

> A Activity启动B Activity的生命周期情况： A执行onCreate，onStart，onResume到可交互状态，此时启动Ｂ Activity，首先将执行A Activity的onpause暂停A Activity执行，然后执行B Activity的onCreate，onStart，onResume到可交互状态，然后再执行A Activity的onStop方法(如果B Activity未完全遮挡A Activity, 将不会回调A Activity的onStop方法)，如果启动B Activity时失败，Ａ Activity将从onPause状态回到onResume执行。

> home键/锁屏生命周期情况: onPause -> onStop

> back键/清除recent生命周期情况: onPause -> onStop -> onDestory

## 3. 重建与恢复

#### **1. 什么是重建？引发重建的场景有哪些？**

正常的生命周期流程，是页面从正常创建到销毁全流程。而重建流程，则是在特定条件下，引发页面被销毁，并再次被创建的流程。**通常是 “系统资源的回收” 或 “配置发生变化” 导致的重建**。

**系统资源回收是指：**

当 App 处于背景模式时，可能因系统内存不足而被回收。

例如按下 Home 键切回桌面，或是接电话跳转到电话程序，都有可能使此前的这个 App 处于背景模式。

**配置发生变化是指：**

当系统配置发生变化时，比如屏幕方向（旋转屏幕）、屏幕大小（折叠屏、卷轴屏）、语言的改变等。



#### 2. 为何要设计出重建的机制？有何好处？

**一来：**&#x5BF9;于资源回收的情况，保存状态并等到使用时再恢复，要比后台存留进程 **所占的资源要小得多**。

**二来：**&#x5BF9;于配置变化的情况，比如当屏幕方向发生变化时，**唯有重建，才有机会加载不同的界面**，特别如果横、竖屏的布局不同的话。

#### 3. onSaveInstanceState方法什么时候会执行？

1. 当用户按下HOME键时。系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity是否会被销毁，故系统会调用onSaveInstanceState，让用户有机会保存某些非永久性的数据。
2. 长按HOME键，选择运行其他的程序时。
3. 按下电源按键（关闭屏幕显示）时。
4. 从activity A中启动一个新的activity时。
5. 屏幕方向切换时，例如从竖屏切换到横屏时。

#### _**4. onCreate()里也有Bundle参数，可以用来恢复数据，它和onRestoreInstanceState有什么区别？**_

因为onSaveInstanceState 不一定会被调用，所以onCreate()里的Bundle参数可能为空，如果使用onCreate()来恢复数据，一定要做非空判断。而onRestoreInstanceState的Bundle参数一定不会是空值，因为它只有在上次activity被回收了才会调用。而且onRestoreInstanceState是在onStart()之后被调用的。有时候我们需要onCreate()中做的一些初始化完成之后再恢复数据，用onRestoreInstanceState会比较方便。onSaveInstanceState调用时机在onStop()前,onRestoreInstanceState的调用时机是在onStart()之后。

**注意：**&#x81EA; API 28 起（CompileSDKVersion、TargetSDKVersion >= 28 并且 Android 系统版本 >= 9.0 的环境下），onSaveInstanceState 执行时机已确定为在 `onStop` 之后。

> _**注意:**_

> onSaveInstanceState方法和onRestoreInstanceState方法“不一定”是成对的被调用的，onRestoreInstanceState被调用的前提是，activity A“确实”被系统销毁了，而如果仅仅是停留在有这种可能性的情况下，则该方法不会被调用，例如，当正在显示activity A的时候，用户按下HOME键回到主界面，然后用户紧接着又返回到activity A，这种情况下activity A一般不会因为内存的原因被系统销毁，故activity A的onRestoreInstanceState方法不会被执行。

> onSaveInstanceState和onRestoreInstanceState方法中，系统自动做了一系列恢复动作，系统会默认保存当前Activity视图结构，并在Activity重启后恢复这些数据。Activity被意外终止时，首先会回调onSaveInstanceState保存数据，然后委托Window -> DecorView -> View一层层保存数据。与Activity一样，每个View都有onSaveInstanceState和onRestoreInstanceState方法。

## 4. 任务栈

`ActivityRecord`：

描述 Activity 的相关信息，对应着一个用户界面，是 Activity 管理的最小单位。

`TaskRecord`：

**是一个栈结构**，管理着多个 ActivityRecord，栈顶的 ActivityRecord 表示当前获得焦点的窗口。

`ActivityStack`：

是一个栈结构，管理着多个 TaskRecord，栈顶的 TaskRecord 表示当前获得焦点的任务。

`ActivityStackSupervisior`：

管理着多个 ActivityStack。**当前只会有一个获得了焦点的 ActivityStack**。

![](<../../.gitbook/assets/image (340).png>)

## 4. 启动模式

### 任务栈(task stack)

为了记录用户开启了那些activity，记录这些activity开启的先后顺序，google引入任务栈（task stack）概念，以栈的结构(先进后出)将依次打开的activity记录保存在task stack中。

### 启动模式

#### Standard

默认Standard模式，假设没有为Activity设置启动模式的话，默认是标准模式。即每次启动一个Activity都会又一次创建一个新的实例入栈，无论这个实例是否存在。Standard模式的Activity默认会进入启动它的Activity所属的任务栈（启动它的Activity为SingleInstance的情况除外）。

#### SingleTop

当创建的Activity已经处于栈顶时，此时会直接复用栈顶的Activity，不会再创建新的Activity。若须要创建的Activity不处于栈顶，此时会又一次创建一个新的Activity入栈，同Standard模式一样。 栈顶的Activity被直接复用时，onCreate、onStart不会被系统调用，而是回调onNewIntent方法（Activity被正常创建时不会回调此方法），然后再执行onResume。(实际是**onPause -> onNewIntent -> onResume**)

#### SingleTask

栈内复用模式，若须要创建的Activity已经处于栈中时，此时不会创建新的Activity，而是将存在栈中的Activity上面的其他Activity所有销毁，使它成为栈顶。 同SingleTop模式中一样，onCreate、onStart不会被系统调用，仅仅回调Activity中的onNewIntent方法。

> 比如启动Activity A，先找其对应的任务栈，如果不存在，则先新建任务栈，在创建Activity A放到任务栈中。如果存在，则判断栈中是否存在Activity A的实例，如果存在则把Activity A调到栈顶，然后调用其onNewIntent方法。如果不存在Activity A的实例，则新建Activity A放到任务栈中。

> SingleTask会导致它之上的栈内所有Activity出栈

#### SingleInstance

具有此模式的Activity仅仅能单独位于一个任务栈中

### taskAffinity

taskAffinity，可以翻译为任务相关性。**这个参数标识了一个 Activity 所需要的任务栈的名字**，默认情况下，所有 Activity 所需的任务栈的名字为应用的包名，当 Activity 设置了 taskAffinity 属性，那么这个 Activity 在被创建时就会运行在和 taskAffinity 名字相同的任务栈中，如果没有，则新建 taskAffinity 指定的任务栈，并将 Activity 放入该栈中。另外，taskAffinity 属性主要和singleTask或者allowTaskReparenting属性配对使用，在其他情况下没有意义。

* **taskAffinity对Standard与SingleTop，SingleInstance没有影响**
* taskAffinity对SingleTask的影响： 以A启动B来说 当A和B的taskAffinity相同时：第一次创建B的实例时，并不会启动新的task，而是直接将B添加到A所在的task；当B的实例已经存在时，将B所在task中位于B之上的全部Activity都移除，B就成为栈顶元素，实现跳转到B的功能。 当A和B的taskAffinity不同时：第一次创建B的实例时，会启动新的task，然后将B添加到新建的task中；当B的实例引进存在，将B所在task中位于B之上的全部Activity都移除，B就成为栈顶元素（也是root Activity），实现跳转到B的功能。

### 使用场景

* Standard 普通情况下使用
* SingleTop 适用于同类型的Activity，例如通知栏消息接收启动的Activity
* SingleTask 主界面
* SingleInstance 适合公开成通用功能的Activity，需要与其他应用共享的界面，例如拨号，闹钟界面等

### Activity Flags

1. FLAG\_ACTIVITY\_NEW\_TASK 作用是为Activity指定 “SingleTask”启动模式。跟在AndroidMainfest.xml指定效果同样。

> **注意**： FLAG\_ACTIVITY\_NEW\_TASK与FLAG\_ACTIVITY\_CLEAN\_TOP才具备AndroidMainfest.xml指定SingleTask同样效果

1. FLAG\_ACTIVITY\_SINGLE\_TOP 作用是为Activity指定 “SingleTop”启动模式，跟在AndroidMainfest.xml指定效果同样。
2. FLAG\_ACTIVITY\_CLEAN\_TOP 具有此标记位的Activity，启动时会将与该Activity在同一任务栈的其他Activity出栈。一般与SingleTask启动模式一起出现。它会完毕SingleTask的作用。但事实上SingleTask启动模式默认具有此标记位的作用
3. FLAG\_ACTIVITY\_EXCLUDE\_FROM\_RECENTS 具有此标记位的Activity不会出如今历史Activity的列表中，使用场景：当某些情况下我们不希望用户通过历史列表回到Activity时，此标记位便体现了它的效果。它等同于在xml中指定Activity的属性：android:excludeFromRecents="trure"

## 5. 相关

1. Activity因系统内存不足被Kill或是因为Crash闪退的异常退出怎么保存数据? Activity被销毁了以后调用了onSaveInstanceState来保存数据，这个只会在Activity即将销毁并且有机会重新显示的情况下才会去调用onSaveInstanceState，与之配对的是在onRestoreInstanceState里恢复数据。
2. 资源不足时Activity的杀死顺序
   * 前台Activity，正在和用户交互的Activity，优先级最高
   * 可见但非前台Activity，比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户直接交互。
   * 后台Activity，已经被暂停的Activity，比如执行了onStop，优先级最低
3. 横竖屏切换时候Activity的生命周期 跟Manifest配置有关
   * 不设置Activity的android:configChanges时，切屏会重新调用各个生命周期默认首先销毁当前activity，然后重新创建
   * 设置Activity的android:configChanges="orientation|screenSize"时，切屏不会重新调用各个生命周期，只会执onConfigurationChanged方法
4. 启动其他应用Activity时，报java.lang.SecurityException: Permission Denial，因为在其他应用StartActivity时，有去检验permission。

> 满足以下条件，可以成功startActivity，不会出现permission denial：
>
> * 同一个application下
> * Uid相同
> * permission匹配
> * 目标Activity的属性Android:exported=”true”
> * 目标Activity具有相应的IntentFilter，存在Action动作或其他过滤器并且没有设置exported=false
> * 启动者的Pid是一个System Server的Pid
> * 启动者的Uid是一个System Uid（android.system.uid=1000）
