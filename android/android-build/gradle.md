# Gradle详解

> 在Java项目中，有两个主要的构建系统：Gradle和Maven。构建系统主要管理潜在的复杂依赖关系并正确编译项目。

### 0. Gradle是什么？

Gradle是一个开源的自动构建工具，它在设计之初就是为了能够灵活地构建几乎所有类型的应用。

构建工具，顾名思义就是用于构建（Build）的工具，构建包括编译（Compile）、连接（Link）、将代码打包成可用或可执行形式等等。

如果不使用构建工具，那么对于开发者而言，下载依赖、将源文件编译成二进制代码、打包等工作都需要一步步地手动完成。但如果使用构建工具，我们只需要编写构建工具的脚本，构建工具就会自动地帮我们完成这些工作。

java生态圈的三大构建工具：

* Ant： 2000年发布，纯java语言编写，具有良好的跨平台性，用buil.xml文件来配置，需要搭配Apache lvy工具来实现网络依赖管理。 Ant是程序式的构建工具，需要自定义构建过程，优点是对于构建过程有良好的控制性 。
* Maven ：2004年发布，对Ant进行了改进，用prom.xml文件来配置。但与Ant不同的是，Maven是申明式的构建工具，对目录结构有约束，不需要自定义构建过程，配置较为简单。Maven还具有生命周期，更重要的是Maven内置了依赖管理 。
* Gradle ：2012年发布，Gradle结合了前两者的优点，在此基础之上做了很多改进。它具有Ant的强大和灵活，又有Maven的生命周期管理且易于使用 。 Gradle不用XML，它使用基于Groovy的专门开发的DSL，所以它的配置文件更加简洁。它跟ant一样,使用了ivy作为jar包的依赖管理工具。

![](<../../.gitbook/assets/image (59).png>)

### 1. Gradle基础

#### **1.1 Gradle环境配置**

* 安装Java环境
* 下载最新 [Gradle发行版](https://gradle.org/releases/)
*   解压并配置环境变量

    ```
    $ mkdir /opt/gradle
    $ unzip -d /opt/gradle gradle-6.0.1-bin.zip
    $ ls /opt/gradle/gradle-6.0.1
    ​
    $ GRADLE_HOME=/opt/gradle/gradle-6.0.1
    $ export GRADLE_HOME
    $ export PATH=${PATH}:${GRADLE_HOME}/bin
    ```

    更多安装指南可参考[官方文档](https://gradle.org/install/)。

#### **1.2 Gradle工程目录**

&#x20;在gradle环境搭建好后，使用`gradle init`命令(Android Studio会自动执行)可初始化一个gradle构建的工程目录，如下所示：

```
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── settings.gradle
```

* build.gralde是Gradle默认的构建脚本文件，执行gradle命令会默认加载当前目录下的build.gradle文件。
*   gradle/wrapper主要用于实现gradle的版本兼容，该目录中包含了gradle-wrapper.jar和gradle－wrapper.properties两个文件，其中jar包就是一个可执行的gradle，gradle-wrapper.properties主要是配置与这个可执行jar包相关的参数。

    ```
    #Wed Jun 12 14:35:57 CST 2019
    distributionBase=GRADLE_USER_HOME  //gradle包下载路径
    distributionPath=wrapper/dists     //gradle.jar存储的相对路径，与前面的基本路径组合成了绝对路径
    zipStoreBase=GRADLE_USER_HOME      //同distributionBase，只存放zip包
    zipStorePath=wrapper/dists         //同distributionPath，只存放zip包
    distributionUrl=https\://services.gradle.org/distributions/gradle-5.4.1-all.zip  //需要下载的对应的gradle发行版下载路径
    ```
* gradlew与gradlew.bat是Unix系统和Window系统下的可执行脚本，用于执行特定版本的gradle（也就是上面gradle文件下的jar包，如果修改gradle-wrapper.properties下对应的属性，重新执行gradlew脚本就会重新下载）。
* settings.gradle用于初始化和配置多项目工程树，存放在项目根目录下。

#### **1.2 Gradle核心概念**

Gradle所有的执行都基于两个基本概念：**poject和task**。每一个Gradle构建都是由一个或多个project组成，每个project都是有一个或者多个task组成。同时每个task之间存在依赖关系，以保证了任务的执行顺序。

**Project**

Project在Android开发里面对应Android Studio中module，每个project都有一个build.gradle配置。在我们Android项目里面，在项目的根目录会有一个build.gradle文件，同时在每个模块的目录下也会有一个build.gradle 文件。

> PS: 实际上在gradle的构建流程中，Gradle会根据build.gradle配置文件创建一个project实例，然后把build.gradle中的配置以task任务的方式插入到project中，执行project实例

&#x20;可以通过`./gradlew projects`指令可以列出当前工程目录下所有的project，例如：

```
> Task :projects
​
------------------------------------------------------------
## Root project
------------------------------------------------------------
​
Root project 'MyApplication'
+--- Project ':app'
+--- Project ':buildSrc'
\--- Project ':mylibrary'
```

&#x20;另外我们常见的如下allprojects(Clouse)配置，就是配置当前project和它的所有子project。

```
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```

**Task**

Task代表构建执行的一些原子工作，比如编译某些类，创建JAR，生成Javadoc或将一些jar或者aar文件发布到maven仓库。可以通过`gradle tasks`指令可以查看当前项目中的所有task。

```
Starting a Gradle Daemon, 1 incompatible Daemon could not be reused, use --status for details
​
> Configure project :OPCommonCtrlLib 
rootProject.libsDirName = libs
​
> Task :tasks 
​
------------------------------------------------------------
All tasks runnable from root project
------------------------------------------------------------
​
Android tasks
-------------
androidDependencies - Displays the Android dependencies of the project.
signingReport - Displays the signing info for each variant.
sourceSets - Prints out all the source sets defined in this project.
​
Build tasks
-----------
assemble - Assembles all variants of all applications and secondary packages.
assembleAndroidTest - Assembles all the Test applications.
assembleDebug - Assembles all Debug builds.
assembleH2 - Assembles all H2 builds.
assembleO2 - Assembles all O2 builds.
assembleRelease - Assembles all Release builds.
build - Assembles and tests this project.
buildDependents - Assembles and tests this project and all projects that depend on it.
buildNeeded - Assembles and tests this project and all projects it depends on.
clean - Deletes the build directory.
cleanBuildCache - Deletes the build cache directory.
compileDebugAndroidTestSources
compileDebugSources
compileDebugUnitTestSources
compileH2DebugAndroidTestSources
compileH2DebugSources
compileH2DebugUnitTestSources
compileH2ReleaseSources
compileH2ReleaseUnitTestSources
compileO2DebugAndroidTestSources
compileO2DebugSources
compileO2DebugUnitTestSources
compileO2ReleaseSources
compileO2ReleaseUnitTestSources
compileReleaseSources
compileReleaseUnitTestSources
extractDebugAnnotations - Extracts Android annotations for the debug variant into the archive file
extractReleaseAnnotations - Extracts Android annotations for the release variant into the archive file
mockableAndroidJar - Creates a version of android.jar that's suitable for unit tests.
​
Build Setup tasks
-----------------
init - Initializes a new Gradle build.
wrapper - Generates Gradle wrapper files.
​
Help tasks
----------
buildEnvironment - Displays all buildscript dependencies declared in root project 'OPFilemanager'.
components - Displays the components produced by root project 'OPFilemanager'. [incubating]
dependencies - Displays all dependencies declared in root project 'OPFilemanager'.
dependencyInsight - Displays the insight into a specific dependency in root project 'OPFilemanager'.
dependentComponents - Displays the dependent components of components in root project 'OPFilemanager'. [incubating]
help - Displays a help message.
model - Displays the configuration model of root project 'OPFilemanager'. [incubating]
projects - Displays the sub-projects of root project 'OPFilemanager'.
properties - Displays the properties of root project 'OPFilemanager'.
tasks - Displays the tasks runnable from root project 'OPFilemanager' (some of the displayed tasks may belong to subprojects).
​
Install tasks
-------------
installDebugAndroidTest - Installs the android (on device) tests for the Debug build.
installH2Debug - Installs the DebugH2 build.
installH2DebugAndroidTest - Installs the android (on device) tests for the H2Debug build.
installO2Debug - Installs the DebugO2 build.
installO2DebugAndroidTest - Installs the android (on device) tests for the O2Debug build.
uninstallAll - Uninstall all applications.
uninstallDebugAndroidTest - Uninstalls the android (on device) tests for the Debug build.
uninstallH2Debug - Uninstalls the DebugH2 build.
uninstallH2DebugAndroidTest - Uninstalls the android (on device) tests for the H2Debug build.
uninstallH2Release - Uninstalls the ReleaseH2 build.
uninstallO2Debug - Uninstalls the DebugO2 build.
uninstallO2DebugAndroidTest - Uninstalls the android (on device) tests for the O2Debug build.
uninstallO2Release - Uninstalls the ReleaseO2 build.
​
Verification tasks
------------------
check - Runs all checks.
connectedAndroidTest - Installs and runs instrumentation tests for all flavors on connected devices.
connectedCheck - Runs all device checks on currently connected devices.
connectedDebugAndroidTest - Installs and runs the tests for debug on connected devices.
connectedH2DebugAndroidTest - Installs and runs the tests for h2Debug on connected devices.
connectedO2DebugAndroidTest - Installs and runs the tests for o2Debug on connected devices.
deviceAndroidTest - Installs and runs instrumentation tests using all Device Providers.
deviceCheck - Runs all device checks using Device Providers and Test Servers.
lint - Runs lint on all variants.
lintDebug - Runs lint on the Debug build.
lintH2Debug - Runs lint on the H2Debug build.
lintH2Release - Runs lint on the H2Release build.
lintO2Debug - Runs lint on the O2Debug build.
lintO2Release - Runs lint on the O2Release build.
lintRelease - Runs lint on the Release build.
lintVitalH2Release - Runs lint on just the fatal issues in the h2Release build.
lintVitalO2Release - Runs lint on just the fatal issues in the o2Release build.
test - Run unit tests for all variants.
testDebugUnitTest - Run unit tests for the debug build.
testH2DebugUnitTest - Run unit tests for the h2Debug build.
testH2ReleaseUnitTest - Run unit tests for the h2Release build.
testO2DebugUnitTest - Run unit tests for the o2Debug build.
testO2ReleaseUnitTest - Run unit tests for the o2Release build.
testReleaseUnitTest - Run unit tests for the release build.
​
Rules
-----
Pattern: clean<TaskName>: Cleans the output files of a task.
Pattern: build<ConfigurationName>: Assembles the artifacts of a configuration.
Pattern: upload<ConfigurationName>: Assembles and uploads the artifacts belonging to a configuration.
​
To see all tasks and more detail, run gradlew tasks --all
​
To see more detail about a task, run gradlew help --task <task>
​
​
BUILD SUCCESSFUL in 5s
1 actionable task: 1 executed
```

Task及依赖

1.  Task创建

    ```
    task customTask {
        doFirst {
            println "doFirst action"
        }
    ​
        doLast{
            println "doLast action"
        }
    }
    ```

    举例：经常使用的打包jar

    ```
    task makeJar(type: Jar) {
        //指定生成的jar名
        baseName = “sdk”
        //从哪里打包class文件
        from zipTree("build/intermediates/packaged-classes/release/classes.jar")
        //打包到哪个位置
        destinationDir = file("build/outputs/jar/")
    } 
    ​
    makeJar.dependsOn(clearJar, build)
    ```

    ./gradlew makeJar 或者在Androd Studio的gradle task界面上手动触发即可执行。
2.  任务依赖denpendsOn

    任务的依赖关系通过dependsOn关键词声明。任务依赖的主要意义在于确定Task与Task之间的执行先后顺序。Task与Task之间构成一个**有向无环图(DAG)**，每个Task最多执行一次。

### 2. Gradle生命周期

![](<../../.gitbook/assets/image (244).png>)

无论什么时候执行 Gradle 构建，都会运行三个不同的生命周期阶段：

1. 初始化阶段
2. 配置阶段
3. 执行阶段

#### 2.1 初始化阶段：

在**初始化阶段**，Gradle 根据 settings.gradle 文件的配置为项目创建了 Project 实例。在给定的构建脚本中只定义了一个项目。在多项目构建中，这个构建阶段变得更加重要。根据你正在执行的项目，Gradle 找出哪些项目需要参与到构建中。实质为执行 settings.gradle 脚本。注意，在这个阶段当前已有的构建脚本代码都不会被执行。\
用户可以在 settings.gradle 文件中调用 Settings 类的各种方法配置项目，最常用的就是 include 方法，它可以将用户新建的module加入项目中。

[Gradle 官方文档：Settings 类](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html#org.gradle.api.initialization.Settings)

#### 2.2 配置阶段：

在**配置阶段**，Gradle 构造了一个模型来表示任务，并参与到构建中来。增量式构建特性决定来模型中的 task 是否需要运行。配置阶段完成后，整个 build 的 project 以及内部的 Task 关系就确定了。这个阶段非常适合于为项目或指定 task 设置所需的配置。配置阶段的实质为解析每个被加入构建项目的 build.gradle 脚本，比如通过 apply 方法引入插件，为插件扩展属性进行的配置等等。\
注意，项目的每一次构建的任何配置代码都可以被执行–即使你只执行 `gradle tasks`。

#### 2.3 执行阶段：

在**执行阶段**，所有的 task 都应该以正确的顺序被执行。执行顺序时由它们的依赖决定的。如果任务被认为没有被修改过，将被跳过。

**Gradle 的增量式的构建特性紧紧地与生命周期相结合。**\
作为一个开发人员，不能仅限于编写在不同构建阶段执行的 task 动作或者配置逻辑。有时候当一个特定的生命周期事件发生时你可能想要执行代码。一个声明周期事件可能发生在某个构建阶段之前、期间或者之后。在执行阶段之后发生的生命周期事件是构建的完成。\
我们有两种方式可以编写回调声明周期事件：在闭包中，或者是通过 Gradle API 所提供的监听器接口实现。Gradle 不会引导你采用哪种方式去监听生命周期事件，着完全取决于你的选择。下面提供一个有用的声明周期钩子（Hook）的用法。

![](<../../.gitbook/assets/image (375).png>)

许多生命周期回调方法被定义在 `Gradle` 和 `Projet` 接口中。\
不要害怕使用生命周期钩子，它们不是 Gradle API 的秘密后门，相反，它们是特意提供给开发者使用的。

### 3. 生命周期监听方法 <a href="#sheng-ming-zhou-qi-jian-ting-fang-fa" id="sheng-ming-zhou-qi-jian-ting-fang-fa"></a>

如果我们想在 Gradle 特定的阶段去 Hook 指定的任务，那么就需要对如何监听生命周期回调做一些了解。`Gradle` 和 `Projet` 对象提供了一些方法来供我们设置一些生命周期的回调方法。\
生命周期监听的设置有两种方法：

1. 实现一个特定的监听接口；
2. 提供一个用于在收到通知时执行的闭包。

上面两个对象对这两种方法的都是支持的。`Projet` 提供的一些生命周期回调方法：

* afterEvaluate(closure)，afterEvaluate(action)
* beforeEvaluate(closure)，beforeEvaluate(action)

`Gradle` 提供的一些生命周期回调方法：

* afterProject(closure)，afterProject(action)
* beforeProject(closure)，beforeProject(action)
* buildFinished(closure)，buildFinished(action)
* projectsEvaluated(closure)，projectsEvaluated(action)
* projectsLoaded(closure)，projectsLoaded(action)
* settingsEvaluated(closure)，settingsEvaluated(action)
* addBuildListener(buildListener)
* addListener(listener)
* addProjectEvaluationListener(listener)

可以看到，每个方法都有两个不同参数的方法，一个接收闭包作为回调，另外一个接受 `Action` 作为回调，下面的介绍时只介绍闭包为参数的方法。\
**请注意**：一些声明周期事件只有在适当的位置上声明才会发生。

下面开始介绍 project 的几个方法：

#### 3.1 beforeEvaluate <a href="#beforeevaluate" id="beforeevaluate"></a>

`beforeEvaluate()`是在 project 开始配置前调用，当前的 project 作为参数传递给闭包。\
这个方法很容易误用，你要是直接当前子模块的 build.gradle 中使用是肯定不会调用到的，因为Project都没配置好所以也就没它什么事情，这个代码块的添加只能放在父工程的 build.gradle 中,如此才可以调用的到。

```
this.project.subprojects { sub ->
    sub.beforeEvaluate { project
        println "#### Evaluate before of "+project.path
    }
}
```

如果你想用 `Action` 作为参数的方法：

```
this.project.subprojects { sub ->
    sub.beforeEvaluate(new Action<Project>() {
        @Override
        void execute(Project project) {
            println "#### Evaluate before of "+project.path
        }
    })
}
```

#### afterEvaluate <a href="#afterevaluate" id="afterevaluate"></a>

`afterEvaluate` 是一般比较常见的一个配置参数的回调方式，只要 project 配置成功均会调用，不论是在父模块还是子模块。参数类型以及写法与afterEvaluate相同：

```
project.afterEvaluate { pro ->
    println("#### Evaluate after of " + pro.path)
}
```

再来看一下 Gradle 对象的几个回调，可以通过 project 获取当前的 gradle 对象，gradle 设置的回调监控的是所有的 project 实现。

#### afterProject <a href="#afterproject" id="afterproject"></a>

设置一个 project 配置完毕后立即执行的闭包或者回调方法。\
afterProject 在配置参数失败后会传入两个参数，前者是当前 project，后者显示失败信息。

```
this.getGradle().afterProject { project,projectState ->
    if(projectState.failure){
        println "Evaluation afterProject of "+project+" FAILED"
    } else {
        println "Evaluation afterProject of "+project+" succeeded"
    }
}
```

#### beforeProject <a href="#beforeproject" id="beforeproject"></a>

设置一个 project 配置前执行的闭包或者回调方法。\
当前 project 作为参数传递给闭包。\
子模块的该方法声明在 root project 中回调才会执行，root project 的该方法声明在 settings.gradle 中才会执行。

```
gradle.beforeProject { p ->
    println("Evaluation beforeProject"+p)
}
```

#### buildFinished <a href="#buildfinished" id="buildfinished"></a>

构建结束时的回调，此时所有的任务都已经执行，一个构建结果的对象 BuildResult 作为参数传递给闭包。

```
gradle.buildFinished { r ->
    println("buildFinished "+r.failure)
}
```

#### projectsEvaluated <a href="#projectsevaluated" id="projectsevaluated"></a>

所有的 project 都配置完成后的回调，此时，所有的project都已经配置完毕，准备开始生成 task 图。gradle 对象会作为参数传递给闭包。

```
gradle.projectsEvaluated {gradle ->
    println("projectsEvaluated")
}
```

#### projectsLoaded <a href="#projectsloaded" id="projectsloaded"></a>

当 setting 中的所有project 都创建好时执行闭包回调。gradle 对象会作为参数传递给闭包。\
这个方法也比较特殊，只有声明在适当的位置上才会发生，如果将这个声明周期挂接闭包声明在 build.gradle 文件中，那么将不会发生这个事件，因为项目创建发生在初始化阶段。\
放在 settings.gradle 中是可以执行的。

```
gradle.projectsLoaded {gradle ->    println("@@@@@@@ projectsLoaded")}
```

#### settingsEvaluated <a href="#settingsevaluated" id="settingsevaluated"></a>

当 settings.gradle 加载并配置完毕后执行闭包回调，setting对象已经配置好并且准备开始加载构建 project。\
这个回调在 build.gradle 中声明也是不起作用的，在 settings.gradle 中声明是可以的。

```
gradle.settingsEvaluated {
    println("@@@@@@@ settingsEvaluated")
}
```

前面我们说过，设置监听回调还有另外一种方法，通过设置接口监听添加回调来实现。作用的对象均是所有的 project 实现。

#### addProjectEvaluationListener <a href="#addprojectevaluationlistener" id="addprojectevaluationlistener"></a>

```
gradle.addProjectEvaluationListener(new ProjectEvaluationListener() {
    @Override
    void beforeEvaluate(Project project) {
        println " add project evaluation lister beforeEvaluate,project path is: "+project
    }
    @Override
    void afterEvaluate(Project project, ProjectState state) {
        println " add project evaluation lister afterProject,project path is:"+project
    }
})
```

#### addListener <a href="#addlistener" id="addlistener"></a>

添加一个实现来 listener 接口的对象到 build。

#### addBuildListener <a href="#addbuildlistener" id="addbuildlistener"></a>

添加一个 `BuildListener` 对象到 Build 。

```
gradle.addBuildListener(new BuildListener() {
    @Override
    void buildStarted(Gradle gradle) {
        println("### buildStarted")
    }
    @Override
    void settingsEvaluated(Settings settings) {
        println("### settingsEvaluated")
    }
    @Override
    void projectsLoaded(Gradle gradle) {
        println("### projectsLoaded")
    }
    @Override
    void projectsEvaluated(Gradle gradle) {
        println("### projectsEvaluated")
    }
    @Override
    void buildFinished(BuildResult result) {
        println("### buildFinished")
    }
})
```

### 4. Task 执行图 <a href="#task-zhi-hang-tu-taskexecutiongraph" id="task-zhi-hang-tu-taskexecutiongraph"></a>

在配置时，Gradle 决定了在执行阶段要运行的 task 的顺序，他们的依赖关系的内部结构被建模为一个有向无环图，我们可以称之为 taks 执行图，它可以用 `TaskExecutionGraph` 来表示。可以通过 `gradle.taskGraph` 来获取。\
在 `TaskExecutionGraph` 中也可以设置一些 Task 生命周期的回调：

* addTaskExecutionGraphListener(TaskExecutionGraphListener listener)
* addTaskExecutionListener(TaskExecutionListener listener)
* afterTask(Action action)，afterTask(Closure closure)
* beforeTask(Action action)，beforeTask(Closure closure)
* whenReady(Action action)，whenReady(Closure closure)

下面来进行详细介绍。

#### addTaskExecutionGraphListener <a href="#addtaskexecutiongraphlistener" id="addtaskexecutiongraphlistener"></a>

添加 task 执行图的监听器，当执行图配置好会执行通知。

```
gradle.taskGraph.addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        println("@@@ gradle.taskGraph.graphPopulated ")
    }
})
```

#### addTaskExecutionListener <a href="#addtaskexecutionlistener" id="addtaskexecutionlistener"></a>

添加 task 执行监听器，当 task 执行前或者执行完毕会执行回调发出通知。

```
gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    @Override
    void beforeExecute(Task task) {
        println("@@@ gradle.taskGraph.beforeTask "+task)
    }
    @Override
    void afterExecute(Task task, TaskState state) {
        println("@@@ gradle.taskGraph.afterTask "+task)
    }
})
```

#### afterTask <a href="#aftertask" id="aftertask"></a>

设置一个 task 执行完毕的闭包或者回调方法。该 task 作为参数传递给闭包。

```
gradle.taskGraph.afterTask { task ->
    println("### gradle.taskGraph.afterTask "+task)
}
```

#### beforeTask <a href="#beforetask" id="beforetask"></a>

设置一个 task 执行前的闭包或者回调方法。该 task 作为参数传递给闭包。

```
gradle.taskGraph.beforeTask { task ->
    println("### gradle.taskGraph.beforeTask "+task)
}
```

#### whenReady <a href="#whenready" id="whenready"></a>

设置一个 task 执行图准备好后的闭包或者回调方法。该 taskGrahp 作为参数传递给闭包。

```
gradle.taskGraph.whenReady { taskGrahp ->
    println("@@@ gradle.taskGraph.whenReady ")
}
```

#### 生命周期顺序

我们通过在生命周期回调中添加打印的方法来看一下他们的执行顺序。为了看一下配置 task 的时机，我们在 app 模块中创建来一个 taks：

```
task hello {
    doFirst {
        println '*** task hello doFirst'
    }
    doLast {
        println '*** task hello doLast'
    }
    println '*** config task hello'
} 
```

为了保证生命周期的各个回调方法都被执行，我们在 settings.gradle 中添加各个回调方法。

```
gradle.addBuildListener(new BuildListener() {
    @Override
    void buildStarted(Gradle gradle) {
        println("### gradle.buildStarted")
    }
    @Override
    void settingsEvaluated(Settings settings) {
        println("### gradle.settingsEvaluated")
    }
    @Override
    void projectsLoaded(Gradle gradle) {
        println("### gradle.projectsLoaded")
    }
    @Override
    void projectsEvaluated(Gradle gradle) {
        println("### gradle.projectsEvaluated")
    }
    @Override
    void buildFinished(BuildResult result) {
        println("### gradle.buildFinished")
    }
})
gradle.afterProject { project,projectState ->
    if(projectState.failure){
        println "### gradld.afterProject "+project+" FAILED"
    } else {
        println "### gradle.afterProject "+project+" succeeded"
    }
}
gradle.beforeProject { p ->
    println("### gradle.beforeProject "+p)
}
gradle.allprojects(new Action<Project>() {
    @Override
    void execute(Project project) {
        project.beforeEvaluate { project
            println "### project.beforeEvaluate "+project
        }
        project.afterEvaluate { pro ->
            println("### project.afterEvaluate " + pro)
        }
    }
})
gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    @Override
    void beforeExecute(Task task) {
        if (task.name.equals("hello")){
            println("@@@ gradle.taskGraph.beforeTask "+task)
        }
    }
    @Override
    void afterExecute(Task task, TaskState state) {
        if (task.name.equals("hello")){
            println("@@@ gradle.taskGraph.afterTask "+task)
        }
    }
})
gradle.taskGraph.addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        println("@@@ gradle.taskGraph.graphPopulated ")
    }
})
gradle.taskGraph.whenReady { taskGrahp ->
    println("@@@ gradle.taskGraph.whenReady ")
}
```

执行 task hello：

```
./gradlew hello
### gradle.settingsEvaluated
### gradle.projectsLoaded
> Configure project : 
### gradle.beforeProject root project 'TestSomething'
### project.beforeEvaluate root project 'TestSomething'
### gradle.afterProject root project 'TestSomething' succeeded
### project.afterEvaluate root project 'TestSomething'
> Configure project :app 
### gradle.beforeProject project ':app'
### project.beforeEvaluate project ':app'
*** config task hello
### gradle.afterProject project ':app' succeeded
### project.afterEvaluate project ':app'
> Configure project :common 
### gradle.beforeProject project ':common'
### project.beforeEvaluate project ':common'
### gradle.afterProject project ':common' succeeded
### project.afterEvaluate project ':common'
### gradle.projectsEvaluated
@@@ gradle.taskGraph.graphPopulated 
@@@ gradle.taskGraph.whenReady 
> Task :app:hello 
@@@ gradle.taskGraph.beforeTask task ':app:hello'
*** task hello doFirst
*** task hello doLast
@@@ gradle.taskGraph.afterTask task ':app:hello'
BUILD SUCCESSFUL in 1s
1 actionable task: 1 executed
### gradle.buildFinished
```

因此，生命周期回调的执行顺序是：\
gradle.settingsEvaluated->\
gradle.projectsLoaded->\
gradle.beforeProject->\
project.beforeEvaluate->\
gradle.afterProject->\
project.afterEvaluate->\
gradle.projectsEvaluated->\
gradle.taskGraph.graphPopulated->\
gradle.taskGraph.whenReady->\
gradle.buildFinished

### 5. Gradle插件

#### 5.1 什么是Gradle插件？

Gradle作为一个自动化构建的工具，只是提供了基本的核心框架功能（可以理解为一个编程框架），Gradle最大特点就是支持通过它提供的api实现各种插件，比如编译Java源码的能力，编译Android工程的能力都是通过实现插件来实现的。

&#x20;最典型的，Android应用开发必须要使用的插件就是`com.android.application`。

```
apply plugin: 'com.android.application'
​
android {
    compileSdkVersion 29
    buildToolsVersion '29.0.2'
​
    defaultConfig {
    }
    
    buildTypes {
    }
    
    sourceSets {
    }
    
    dependencies {
    }
​
    compileOptions {
    }
}
```

在Gradle中一般有两种类型的插件，**脚本插件**和**对象插件**。

&#x20;**脚本插件**：脚本插件是额外的构建脚本，它会进一步配置构建，可以把它理解为一个普通的build.gradle，比如在config.gradle定义版本信息，然后在build.gradle中来应用这个插件。

```
>>> config.gradle
ext {
    android = [
            compileSdkVersion: 29,
            buildToolsVersion: "29.0.2",
            minSdkVersion    : 29,
            targetSdkVersion : 29,
            versionCode      : 1,
            versionName      : "1.0",
    ]
​
    appcompatVersion = "1.0.2"
    constraintlayoutVersion = "1.1.3"
    junitVersion = "4.12"
    runnerVersion = "1.1.1"
    espressoVersion = "3.1.1"
    ......
}
​
>>> build.gradle
​
apply from: 'config.gradle'
```

&#x20;**对象插件:** 对象插件就是实现了org.gradle.api.plugins\<Project>接口的插件，对象插件可以分为内部插件和第三方插件。Gradle 的发行包中有大量的内部插件，比如语言插件、集成插件、软件开发插件。

&#x20;内部插件举例：

```
apply plugin: 'java'
apply plugin: 'cpp'
```

&#x20;**第三方插件:** 第三方的对象插件通常是jar文件，要想让构建脚本知道第三方插件的存在，需要使用buildscrip来设置。

```
apply plugin: 'com.alibaba.arouter'
​
buildscript {
    repositories {
        jcenter()
    }
​
    dependencies {
        classpath "com.alibaba:arouter-register:?"
    }
}
```

#### 5.2 自定义Gradle插件可以用来做什么？

自定义Gradle插件一般在三方框架用得比较广泛，比如低侵入sdk，代码注入，代码解耦。在项目构建过程中实现定制化操作，用很少的代码量就可以完成很多重复或者运行时代码无法处理的工作。

比如：

* 提升工作效率。比如，提高编译打包效率，定制编译流程的操作。
* 三方sdk，无痕埋点，低入侵集成三方库
* 流行的开源库，组件化、插件化，代码解耦，代码注入。

#### 5.3 如何自定义Gradle插件？

实际上自定义插件就是对编译流程进行hook执行自己的定制化操作，Gradle构建过程有三个阶段。

* 初始化（Initialization）\
  Gradle可以构建一个和多个项目。在初始化阶段，Gradle会确定哪些项目参与构建，并且为这些项目创建一个Project实例。
* 配置（Configuration）\
  在这个阶段，会配置project对象。将执行构建的所有项目的构建脚本。也就是说，会执行每个项目的build.gradle文件。
* 执行（Execution）\
  Gradle确定要在执行期间创建和配置的任务子集。子集由传递给gradle命令和当前目录的任务名称参数确定。 Gradle然后执行每个选定的任务。

![](https://github.com/RobinHeZtto/Resource/blob/master/blog/image/gradle/gradle-2.png?raw=true)

自定义插件分为3种方式，主要实现原理都是hook编译流程。

|             |                                                                                     |
| ----------- | ----------------------------------------------------------------------------------- |
| Build       | 把插件写在 build.gradle 文件中，一般用于简单的逻辑，只在该 build.gradle 文件中可见。                            |
| 独立项目        | 一个独立的 Groovy/Java 项目，可以把这个项目打包成 Jar包，可以将Jar发布到托管平台上，供其他人使用。                         |
| buildSrc 项目 | 将插件源代码放在 rootProjectDir/buildSrc/src/main/groovy 中，只对该项目中可见，适用于逻辑较为复杂，但又不需要外部可见的插件。 |

* Build script， 即在项目中的build.gradle文件里添加groovy代码并引用； 确定是只有当前工程可用；
*   buildSrc project， Android Studio会找rootPrjectDir/buildSrc/src/main/groovy目录下的代码。

    ```
    apply plugin: 'groovy'

    repositories {
        google()
        jcenter()
        mavenCentral()
    }

    dependencies {
        implementation gradleApi()
        implementation localGroovy()
        implementation 'com.android.tools.build:gradle:4.1.0'
    }
    ```
* Standalone project， 即单独工程，这也是最常见的方式；一般为传版本到jcenter或maven里。 实现方式很简单， 我们可以在buildSrc里调试， 待需要发版时使用Android Studio打开buildSrc模块就新生成一个工程了

**自定义Gradle plugin:**

1. 首先在AS中新建 `Java or Kotlin Library` , 并命名为`buildSrc` , 注意名字必须为`buildSrc`

然后在Settings.gradle中移除`include ':buildSrc'` , 否则将出现如下错误导致sync失败

```
'buildSrc' cannot be used as a project name as it is a reserved name
```

新建以后的工程如下:

![](<../../.gitbook/assets/image (380).png>)

2\. Sync成功以后, 如下配置`build.gradle`

```
apply plugin: 'groovy'

repositories {
    google()
    jcenter()
    mavenCentral()
}

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation 'com.android.tools.build:gradle:4.1.0'
}
```

3\. 新建MyPlugin文件

```
package com.example.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project target) {
        println("MyPlugin apply")
    }
}
```

4\. 在`main`路径下创建`/resources/META-INF/gradle-plugins` 目录, 然后新建`my-plugin.properties` 文件(文件名可自定义, 但必须以properties结尾), 文件内容如下:

```
implementation-class=com.example.plugin.MyPlugin
```

buildSrc模块结构如下所示:

![](<../../.gitbook/assets/image (36).png>)

5\. 在其他模块中应用自定义插件

![](<../../.gitbook/assets/image (373).png>)

可以看到再编译时, buildSrc将最先开始编译

```
> Task :buildSrc:compileJava NO-SOURCE
> Task :buildSrc:compileGroovy UP-TO-DATE
> Task :buildSrc:processResources UP-TO-DATE
> Task :buildSrc:classes UP-TO-DATE
> Task :buildSrc:jar UP-TO-DATE
> Task :buildSrc:assemble UP-TO-DATE
> Task :buildSrc:compileTestJava NO-SOURCE
> Task :buildSrc:compileTestGroovy NO-SOURCE
> Task :buildSrc:processTestResources NO-SOURCE
> Task :buildSrc:testClasses UP-TO-DATE
> Task :buildSrc:test NO-SOURCE
> Task :buildSrc:check UP-TO-DATE
> Task :buildSrc:build UP-TO-DATE

> Task :app:preBuild UP-TO-DATE
```
