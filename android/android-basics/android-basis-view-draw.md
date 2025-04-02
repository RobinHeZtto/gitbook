# Android基础之绘制流程全解析

## View基础

![](<../../.gitbook/assets/image (303).png>)

## View绘制流程

![](<../../.gitbook/assets/image (90).png>)

如上图所示：`ViewRootImpl` 是连接 `WindowManager` 和 `DecorView` 的纽带，测量、放置和绘制三大流程都是通过 ViewRootImpl 实现的。

在 `ActivityThread` 的 handleResumeActivity() 方法中，会调用 WindowManager 的 addView() 方法，而具体添加 DecorView 的操作是在 `WindowManagerGlobal` 中。在 WindowManagerGlobal 的 addView() 方法中，会把 DecorView 添加到 Window 中，同时会创建 ViewRootImpl ，并调用 ViewRootImpl 的 setView() 方法 把 ViewRootImpl 和 DecorView 关联起来。

View 的绘制流程是从 ViewRootImpl 的 `performTraversals()` 方法开始的，它经过测量（measure）、放置（layout）和绘制（draw）三个过程才能把一个 View 绘制出来，measure() 方法用于测量 View 的宽高，layout() 用于确定 View 在父容器中的放置位置，draw() 负责做具体的绘制操作。

![](<../../.gitbook/assets/image (355).png>)

View 绘制主要的三个方法就是 `onMeasure()`、 `onLayout()`、`onDraw()`，这三个方法要解决的问题就是`画多大`、`在哪画`、`画什么`。

ViewRootImpl 的 performTraversal() 方法会依次调用 `performMeasure()`、`performLayout()` 和 `performDraw()` 三个方法，这三个方法分别完成 DecorView 的测量、放置和绘制三大流程。

performMeasure() 方法会调用 DecorView 的 measure() 方法，在 measure() 方法中又会调用自己的 onMeasure() 方法。

DecorView 的 onMeasure() 方法会调用父类 FrameLayout 的 onMeasure() 方法，在 FrameLayout 的 onMeasure() 方法中，会调用子元素的 onMeasure() 方法测量子元素的宽高，接着子元素会重复父容器的 measure 过程，如此反复完成整个 View 树的遍历。

而 performLayout() 和 performDraw() 的执行流程与 performMeasure() 是类似的。

measure 过程决定了 View 的宽高，layout 过程决定了 View 的四个顶点的坐标和**实际的 View 宽高**，draw 过程则决定了 View 的具体绘制操作，只有 draw() 方法完成后 View 的内容才会在屏幕上展示。

#### Activity 视图层级结构

假如我们有一个继承了 AppCompatActivity 的 MainActivity，并且 activity\_main 布局的内容如下。

![](<../../.gitbook/assets/image (27).png>)

我们现在能感知到的视图层级是下面这样的。

![](<../../.gitbook/assets/image (72).png>)

当我们在 MainActivity 中调用父类的 setContentView() 后，AppCompatActivity 会调用 `AppCompatDelegateImpl` 的 setContentView() 方法，AppCompatDelegateImpl 在这个方法中会把 RelativeLayout 添加到 id 为 `content` 的 ViewGroup 中。

![](<../../.gitbook/assets/image (42).png>)

其中 ContentFrameLayout 也就是 id 为 content 的 ViewGroup 。

![](<../../.gitbook/assets/image (359).png>)

`ensureSubDecor()` 方法会在 subDecor 没有初始化时用 `createSubDecor()` 方法创建 subDecor ，createSubDecor() 方法会调用 Window 的 setContetnView() 方法，把 `abc_screen_toolbar` 布局设为 Window 的内容视图，而这里的 mHasActionBar 只有在 feature 为 `FEATURE_SUPPORT_ACTION_BAR` 时才会为 true。

abc\_screen\_toolbar 布局的内容如下。\


![](<../../.gitbook/assets/image (92).png>)

把 RelativeLayout 放到 mSubDecor 中后，视图层级就变成下面这样了。

![](<../../.gitbook/assets/image (347).png>)

Window 的实现类为 `PhoneWindow`，在 PhoneWindow 的 setContentView() 方法中，会调用 installDecor() 方法创建 DecorView ，然后调用 LayoutInflate 的 inflate() 方法把 ActionBarOverlayLayout 加入到 DecorView 中。

![](<../../.gitbook/assets/image (201).png>)

在 installDecor() 方法中，会调用 generateLayout() 方法生成 mContentParent。

在 `generateLayout()` 方法中，会根据不同的 feature 来生成不同的 DecorView，比如没有设定任何 feature 时，对应的 DecorView 的布局就是 `screen_simple` 。

screen\_simple 布局的实现如下。

![](<../../.gitbook/assets/image (307).png>)

前面的布局加入到 screen\_simple 中后，视图层级就是下面这样的。

![](<../../.gitbook/assets/image (48).png>)

这里的 `action_mode_bar_stub` 是用来显示 ActionMode 的，而 FrameLayout 就是 ID\_ANDROID\_CONTENT 对应的 ViewGroup。

到这里好像还是少了点什么，状态栏哪去了？

![](<../../.gitbook/assets/image (337).png>)

根据 Layout Inspector 的分析，LinearLayout 下面还有一个 id 为 statusBarBackground 的 View ，根据这个 id 在 DecorView 中找到了对应的 mStatusColorViewState 。

![](<../../.gitbook/assets/image (368).png>)

而在 DecorView 的 updateColorViewInt() 方法中，则把状态栏通过 addView() 方法添加到了 DecorView 中。

该方法的调用时序图如下。

![](<../../.gitbook/assets/image (31).png>)

也就是完整的 DecorView 视图层次如下。

![](<../../.gitbook/assets/image (378).png>)

上图对应的 View 树如下。

![](<../../.gitbook/assets/image (338).png>)

#### 测量规格 MeasureSpec

按注释来说，`MeasureSpec 封装了从父 View 传给子 View 的布局要求`，MeasureSpec 在很大程度上决定了一个 View 的尺寸规格，具体的尺寸会受到父容器的影响，因为父容器影响 View 的 MeasureSpec 的创建过程。

在测量过程中，系统会把 View 的 LayoutParams 根据父容器设定的规则转换为对应的 MeasureSpec，然后再根据这个 MeasureSpec 测量出 View 的宽高。

要注意的是，这里的说的宽高是测量宽高，不一定是 View 的最终宽高，原因后面会讲到。

MeasureSpec 代表一个 32 位 int 值，高 2 位代表测量模式 `SpecMode`，低 30 位代表规格大小 `SpecSize`，MeasureSpec 通过把 SpecMode 和 SpecSize 打包成一个 int 值避免过多的对象内存分配，

MeasureSpec 中定义了下面三种测量规格。

![](<../../.gitbook/assets/image (88).png>)

*   待定 UNSPECIFIED

    表示父 View 对子 View 的大小不做限制；
*   精确 EXATCTLY

    父 View 计算好了子 View 具体的宽高，子 View 的最终大小就是 SpecSize 指定的值；
*   最多 AT\_MOST

    父 View 指定了一个可用大小，View 的大小不能大于这个值；

MeasureSpec 用来打包 SpecMode 和 SpecSize 的方法是 makeMeasureSpec() ，代码如下。

![](<../../.gitbook/assets/image (230).png>)

#### MeasureSpec 与 LayoutParams 的关系

在 View 测量时，系统会把 LayoutParams 在父 View 的约束下，转换成对应的 MeasureSpec，然后再根据这个 MeasureSpec 确定 View 测量后的宽高，要靠 LayoutParams 和父 View 一起才能决定子 View 的 测量模式。

DecorView 的测量规格由窗口的尺寸和其 LayoutParams 共同确定，而普通 View 的测量规格由父 View 的 MeasureSpec 和自身的 LayoutParams 决定，MeasureSpec 确定后，就可以在 onMeasure() 方法中确定 View 的测量宽高。

在 ViewRootImpl 的 performTraversals() 方法中，有一段调用 measureHierarchy() 方法的代码，也就是传给 measureHierarchy() 的大小为屏幕尺寸。\


![](https://upload-images.jianshu.io/upload_images/2004563-58c2583a4f1e8f77.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

measureHierarchy() 方法是用来设定子 View ，也就是 DecorView 的大小的。

![](https://upload-images.jianshu.io/upload_images/2004563-f54b7d75cb9f7263.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

measureHierarchy() 中的的 childWidthMeasureSpec 和 childHeightMeasureSpec 就是 DecorView 的测量规格 MeasureSpec。

![](https://upload-images.jianshu.io/upload_images/2004563-f3a0c461b62b2a45.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

通过上面代码可以看出，DecorView 会根据 LayoutParams 中的宽高来设定宽高测量规格。

*   MATCH\_PARENT

    精确模式，DecorView 大小就是窗口大小；
*   WRAP\_CONTENT

    最大模式，大小不定，但是不能超过窗口大小；
*   固定大小

    精确模式，大小为 LayoutParams 中指定的大小；

对于普通 View 来说，View 的 measure 过程由 ViewGroup 传递而来，而 ViewGroup 是在 measureChildWithMargins() 方法中确定子 View 的测量规格的。

![](https://upload-images.jianshu.io/upload_images/2004563-2889ddf4a70e2019.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

下面是 ViewGroup 的 getChildMeasureSpec() 方法获取子 View 的测量规格的方式。

![](https://upload-images.jianshu.io/upload_images/2004563-116b0424dd33d0f2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

其中一段代码如下。

![](https://upload-images.jianshu.io/upload_images/2004563-ba23164dcbc1a174.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

上面这段代码中的 size 是去掉了 padding 后的 size。

这里要注意的是，不是所有 ViewGroup 都会用这样的方式决定子 View 的测量规格，比如 RelativeLayout 用的就是不一样的测量规格。

#### View 测量过程

对于 ViewGroup，除了要完成自己的测量，还要遍历调用子元素的 measure() 方法，而 View 只需要通过 measure() 方法就能确定测量规格。

View 的测量过程由 View 的 measure() 方法完成，measure() 方法是一个 final 类型的方法，子类不能重写。

View 的 measure() 方法会调用 onMeasure() 方法，这个方法我们是可以重写的，onMeasure() 的实现如下。

![](https://upload-images.jianshu.io/upload_images/2004563-5244988c9f7cd145.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

widthMeasureSpec 和 heightMeasureSpec 是从父 View 传过来的宽高测量规格，getDefaultSize() 方法是用来获取默认宽高的，getDefaultSize() 的实现如下。

![](https://upload-images.jianshu.io/upload_images/2004563-d10ce54f90b65087.png?imageMogr2/auto-orient/strip|imageView2/2/w/1034/format/webp)

从 getDefaultSize() 方法中可以看出，当测量模式为 UNSPECIFIED 时，宽/高就是最小宽/高，当测量模式为 AT\_MOST 或 EXACTLY 时，宽/高就是 ViewGroup 指定的 SpecSize。

View 的宽/高由 specSize 决定，直接继承 View 的自定义控件需要重写 onMeasure() 方法并设置 wrap\_content 时的自身大小，否则咋布局中使用 wrap\_content 相当于使用 match\_parent 。

从前面的代码可以了解到，如果 View 在布局中使用 wrap\_content，那么它的 specMode 是 AT\_MOST 模式，这时它的宽/高为 specSize ，这时 View 的 specSize 为 ViewGroup 的 specSize。

比如 activity\_main 的布局是下面这样的。

![](https://upload-images.jianshu.io/upload_images/2004563-8b279acb37bed389.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

那么 MyView 测量后的大小就是 600 ，这个 600 是 dp 换算为 px 后的值。

![](https://upload-images.jianshu.io/upload_images/2004563-3fba6c36f13fb2c8.png?imageMogr2/auto-orient/strip|imageView2/2/w/465/format/webp)

ViewGroup 的 SpecSize 是自身剩余的空间大小，也就是默认子 View 的宽/高为父 View 的剩余控件大小，相当于为宽/高设定的 wrap\_content 无效，变成了 match\_parent 。

如果我们不想让自定义 View 在宽/高设为 wrap\_content 时与父 View 的大小一致，那我们可以像下面这样设定自己的计算好的默认宽/高。

![](https://upload-images.jianshu.io/upload_images/2004563-9a5f00417b8efe35.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

下面来看下 ViewGroup 的测量过程。

ViewGroup 是一个抽象类，没有定义测量的的具体过程，具体的测量过程需要子类实现，下面以 LinearLayout 为例，看一下它的 onMeasure() 方法的实现。

![](https://upload-images.jianshu.io/upload_images/2004563-73f7fe9fee27640e.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

LinearLayout 会根据我们设定的方向设定子 View 的测量规格，下面来看下 measureVertical() 的实现。

![](https://upload-images.jianshu.io/upload_images/2004563-c8717ea560a1ea21.png?imageMogr2/auto-orient/strip|imageView2/2/w/1120/format/webp)

在 measureVertical() 方法中，把每一个子元素都传给了 measureChildBeforeLayout() ，而 measureChildBeforeLayout() 只是调用了 ViewGroup 的 measureChildWithMargin() 方法。

![](https://upload-images.jianshu.io/upload_images/2004563-f1aed81ccedf6f1a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1118/format/webp)

#### View 放置过程

layout() 方法的作用是 ViewGroup 用于确定子元素的位置，当 ViewGroup 的位置确定后，会在 onLayout() 方法中遍历所有子 View 并调用子 View 的 layout() 方法。

layout() 方法用于确定 View 自己的位置，而 onLayout() 方法则用于确定所有子元素的位置，View 的 layout() 方法的实现如下。

![](https://upload-images.jianshu.io/upload_images/2004563-8dbf5878e474af96.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

View 的 layout() 方法首先会通过 setFrame() 方法设定 View 的边框，也就是 mLeft、mRight、mTop 和 mBottom 四个顶点的值，这时 View 在父 View 中的位置就确定了。

![](https://upload-images.jianshu.io/upload_images/2004563-e904f1828c95a4ff.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

设定了四个顶点后，layout() 方法就会调用 onLayout() 方法确定子 View 的位置，View 和 ViewGroup 都没有实现 onLayout() 方法，下面以 LinearLayout 为例，看下 LinearLayout 的 onLayout() 方法的实现。

LinearLayout 的 onLayout() 方法会根据不同的排列方向调用不同的放置方法，当方向为 VERTICAL 时，对应的放置方法为 layoutVertical() ，下面来看下 layoutVertical() 方法的实现。

![](https://upload-images.jianshu.io/upload_images/2004563-a6808a5c10f91c46.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

layoutVertical() 方法会遍历所有子 View 并调用 setChildFrame() 方法指定子 View 的边框（frame）在哪个位置，而 setChildFrame() 方法只是简单调用了 子 View 的 layout() 方法。

childTop 的值会逐渐增加，下一个子 View 的 top 为上一个子 View 的 bottom，也就是排列方向为 VERTICAL 的 LinearLayout 的特性。

#### View 绘制过程

View 绘制分为下面 6 步：

1. 绘制背景
2. 保存 Canvas 图层为后续淡出做准备（可选）
3. 绘制 View 的内容
4. 绘制子 View (dispatchDraw)
5. 绘制淡出边缘并恢复 Canvas 图层（可选）
6. 绘制装饰（比如 foreground 和 scrollbar）

一般情况下第 2 步和第 5 步是不执行的。

![](https://upload-images.jianshu.io/upload_images/2004563-3e17c87d8b0ed9d3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1120/format/webp)

下面来看下绘制相关方法的实现。

![](https://upload-images.jianshu.io/upload_images/2004563-6b4787c3f23155e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/788/format/webp)

drawBackground() 方法首先会通过 Drawable 的 setBounds() 方法设置背景绘制的范围，然后如果我们调用过 scrollTo() 方法，那么 drawBackground() 就会把画布平移到指定位置后再绘制。

View 和 ViewGroup 没有实现 onDraw() 方法，接下来就是 dispatchDraw() 方法，View 没有实现这个方法，下面来看下 ViewGroup 的 dispatchDraw() 方法的实现。

![](https://upload-images.jianshu.io/upload_images/2004563-cf5773348529264c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

在 ViewGroup 的 dispatchDraw() 方法中，首先会调用 buildOrderedChildList() 方法获取子 View 列表，然后遍历子 View ，通过 drawChild() 方法调用每一个子 View 的 draw() 方法。

而第 6 步 drawForegounrd() 只是获取 foreground 对应的 Drawable 并调用它的 draw() 方法。

#### 总结

根据前面讲解的内容，从 ViewRootImpl 的 performTraversals() 方法开始，大致的方法调用时序图如下。

![](https://upload-images.jianshu.io/upload_images/2004563-8b28787359d6e44b.png?imageMogr2/auto-orient/strip|imageView2/2/w/680/format/webp)

View 绘制的三大过程分别是测量、放置和绘制，对应的的三个方法为 onMeasure() 、onLayout() 和 onDraw() 。

测量过程中最重要的就是理解 MeasureSpec 以及自定义 View 时要重写 onMeasure() 方法设置默认宽高。

MeasureSpec 由测量模式 SpecMode 和 SpecSize 组成，SpecMode 分为待定（UNSPECIFIED）、精确（EXACTLY）和最大（AT\_MOST）。

放置过程中最关键的方法就是 setFrame() ，这个方法会把父 View 在 onLayout() 方法中计算好的四个顶点的值赋值给 mTop、mLeft 、mRight 和 mBottom 。

绘制过程的 draw() 方法中主要的 4 个绘制步骤为：绘制背景、绘制 View 内容、绘制子 View 内容以及绘制装饰。

