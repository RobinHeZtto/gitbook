# Jetpack

> Jetpack出现的背景：Android碎片化严重（设备繁多，品牌众多，版本各异，分辨率不统一），项目使用的三方库众多，各种架构体系MVP，MVVM等，很多时候都是由开发者各自完成。

## 1. **什么是Jetpack？**

Jetpack 是Google推出的一个由多个库组成的套件，**是Android基础支持库SDK以外的部分，**&#x53EF;帮助开发者**遵循最佳做法**，**减少样板代码**并**编写可在各种 Android 版本和设备中一致运行的代码**，让开发者精力集中编写重要的代码。

**Jetpack与Androidx的关系：**

Jetpack 中所有库都使用 AndroidX 作为包名，Google把 AndroidX 作为一个开发、测试和发布 Jetpack 库的开源工程。

**Jetpack与Support Library的关系：**

在 2018 Google I/O 大会上，Google宣布了把 `Support Library` 重构至 `AndroidX` 命名空间的计划。在 `Support Library 28`完成了重构，并且发布了 `AndroidX 1.0`。至此，`Support Library` 已经完成了它的历史使命，28 将会是它的最后发布版，并且Google不会继续在 `Support Library` 中修复 bug 或发布新功能。Androidx软件包完全取代了支持库，不仅提供与支持库同等的功能，而且还提供了新的库。

* AndroidX 中的所有软件包都使用一致的命名空间，以字符串 `androidx` 开头。支持库软件包已映射到对应的 `androidx.*` 软件包。
* 与支持库不同，`androidx` 软件包会单独维护和更新。从版本 1.0.0 开始，`androidx` 软件包使用严格的[语义版本控制](https://semver.org/)。
* [版本 28.0.0](https://developer.android.google.cn/topic/libraries/support-library/revisions#28-0-0) 是`Support Library`的最后一个版本。Google将不再发布 `android.support` 库版本。 所有新功能都将在 `androidx` 命名空间中开发。

**Android Jetpack特点：**

1. 加速开发\
   组件可单独使用，也可以协同工作，当使用kotlin语言特性时，可以提高效率，并且具有非常好的向下兼容性。
2. 消除样板代码\
   Android Jetpack可以很方便的管理繁琐的Activity（如后台任务、导航和生命周期管理）。
3. 构建高质量的强大应用\
   Android Jetpack组件围绕现代化设计实践构建而成，具有向后兼容性，可以有效减少崩溃和内存泄漏。

**Androidx源码：**

[https://github.com/androidx/android](https://github.com/androidx/androidx)

## 2. Jetpack发展历程

**Google IO 2017**，Google 推出Architecture Component，包括Lifecycles，LiveData，ViewModel，Room等组件，以帮助开发者设计出稳健的，可测试的，架构清晰的app。

* [Architecture components - introduction (Google I/O '17)](https://www.youtube.com/watch?v=FrteWKKVyzI\&ab_channel=AndroidDevelopers)
* [Architecture components - solving the lifecycle problem (Google I/O '17)](https://www.youtube.com/watch?v=bEKNi1JOrNs\&ab_channel=AndroidDevelopers)
* [Architecture components - persistence and offline (Google I/O '17)](https://www.youtube.com/watch?v=MfHsPGQ6bgE\&ab_channel=AndroidDevelopers)

**Google IO 2018，**&#x47;oogle将Support lib更名为androidx，将许多Google认为是正确的方案和实践集中起来，以高效的开发Android APP。

* [Android Jetpack: What’s new in Android Support Library (Google I/O 2018)](https://www.youtube.com/watch?v=jdKUm8tGogw\&ab_channel=AndroidDevelopers)
* [Modern Android development: Android Jetpack, Kotlin, and more (Google I/O 2018)](https://www.youtube.com/watch?v=IrMw7MEgADk\&t=1s\&ab_channel=AndroidDevelopers)
* [Android Jetpack: Manage UI navigation with navigation controller (Google I/O '18)](https://www.youtube.com/watch?v=8GCXtCjtg40\&ab_channel=AndroidDevelopers)
* [Android Jetpack: Manage infinite lists with RecyclerView and Paging (Google I/O '18)](https://www.youtube.com/watch?v=BE5bsyGGLf4\&ab_channel=AndroidDevelopers)
* [Android Jetpack: How to smartly use fragments in your UI (Google I/O '18)](https://www.youtube.com/watch?v=WVPH48lUzGY\&ab_channel=AndroidDevelopers)

**Google IO 2019**，大会上公布新的安卓UI库Jetpack Compose，更多组件迭代。

* [Declarative UI patterns (Google I/O'19)](https://www.youtube.com/watch?v=VsStyq4Lzxo\&ab_channel=AndroidDevelopers)
* [Build a modular Android app architecture (Google I/O'19)](https://www.youtube.com/watch?v=PZBg5DIzNww\&t=178s\&ab_channel=AndroidDevelopers)
* [What's new in architecture components (Google I/O'19)](https://www.youtube.com/watch?v=Qxj2eBmXLHg\&ab_channel=AndroidDevelopers)
* [Jetpack Navigation (Google I/O'19)](https://www.youtube.com/watch?v=JFGq0asqSuA\&ab_channel=AndroidDevelopers)

**2020：**

* [Android Jetpack的新功能](https://www.youtube.com/watch?v=R3caBPj-6Sg\&feature=youtu.be\&ab_channel=AndroidDevelopers)
* [Jetpack 组合编写](https://www.youtube.com/watch?v=U5BwfqBpiWU\&ab_channel=AndroidDevelopers)

## 3. Jetpack组成

Jetpack其最核心的出发点就是帮助开发者快速构建出**稳定、高性能、测试友好**同时**向后兼容**的APP。Jetpack采用的是**组件化**的方式，这样的好处就是每个组件都是相对独立的，也就是说每个组件都是可以被**单独使用和构建**的。 我们的使用也十分的灵活，可以根据我们自己的项目选择我们想要的功能或者是适于我们应用程序的功能。

Android Jetpack 组件将现有的支持库与架构组件联系起来，并将它们分成四个类别：

* **Foundation：基础**&#x20;
* **Architecture：体系结构**
* **UI：视觉交互**&#x20;
* **Behavior：行为**

![](<../../.gitbook/assets/image (125).png>)

**Architecture Compinents（架构组件）：**

* Data Bingding（数据绑定）
* Room（数据库）
* WorkManager（后台任务管家）
* Lifecycle（生命周期）
* Navigation（导航）
* Paging（分页）
* Data Binding（数据绑定）
* LiveData（底层数据通知更改视图）
* ViewModel（以注重生命周期的方式管理界面的相关数据）
* ...

**Foundation（基础）：**

* AppCompat（向后兼容）
* Android KTX（编写更加简洁的Kotlin代码）
* Multidex （多处理dex的问题）
* Test（测试）
* ...

**Behavior（行为）：**

* Download manager（下载给管理器）
* Media & playback（媒体和播放）
* Notifications（通知）
* Permissions（权限）
* Preferences（偏好设置）
* Sharing（共享）
* Slices（切片）
* ...

**UI(视觉交互)**

* Animation & transitions（动画和过渡）
* Auto（Auto组件）
* Emoji（标签）
* Fragment（Fragment）
* Layout（布局）
* Palette（调色板）
* TV（TV）
* Wear OS by Google（穿戴设备）
* ...

## 4. 架构组件

[https://developer.android.google.cn/jetpack/guide#best-practices](https://developer.android.google.cn/jetpack/guide#best-practices)

#### 分离关注点 <a href="#separation-of-concerns" id="separation-of-concerns"></a>

一种常见的错误是在一个 [`Activity`](https://developer.android.google.cn/reference/android/app/Activity) 或 [`Fragment`](https://developer.android.google.cn/reference/android/app/Fragment) 中编写所有代码。这些基于界面的类应**仅包含处理界面和操作系统交互的逻辑**。您应使这些类尽可能保持精简，这样可以避免许多与生命周期相关的问题。

请注意，您并非拥有 `Activity` 和 `Fragment` 的实现；它们只是表示 Android 操作系统与应用之间关系的粘合类。操作系统可能会根据用户互动或因内存不足等系统条件随时销毁它们。为了提供令人满意的用户体验和更易于管理的应用维护体验，您最好尽量减少对它们的依赖。

#### 通过模型驱动界面 <a href="#drive-ui-from-model" id="drive-ui-from-model"></a>

另一个重要原则是您应该**通过模型驱动界面**（最好是持久性模型）。模型是负责处理应用数据的组件。它们独立于应用中的 [`View`](https://developer.android.google.cn/reference/android/view/View) 对象和应用组件，因此不受应用的生命周期以及相关的关注点的影响。

持久性是理想之选，原因如下：

* 如果 Android 操作系统销毁应用以释放资源，用户不会丢失数据。
* 当网络连接不稳定或不可用时，应用会继续工作。

应用所基于的模型类应明确定义数据管理职责，这样将使应用更可测试且更一致。

模块之间的交互，每个组件仅依赖于其下一级的组件。

![](<../../.gitbook/assets/image (465).png>)

## **5. 参考**

* [Android Jetpack](https://developer.android.google.cn/jetpack/getting-started)
* [应用架构指南](https://developer.android.google.cn/jetpack/guide#best-practices)
