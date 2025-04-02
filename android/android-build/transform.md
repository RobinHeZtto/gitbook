# Transform详解

## 1. 什么是Transform

自从`1.5.0-beta1`版本开始, android gradle插件就包含了一个`Transform API`, 它允许第三方插件在编译过程中class转dex之前做处理操作.

`Transform API`, 的目标就是简化class文件的自定义的操作而不用对Task进行处理, 它让我们可以只聚焦在如何对输入的类文件进行处理.

![](<../../.gitbook/assets/image (362).png>)

每个Transform其实都是一个`gradle task`，Android编译器中的`TaskManager`将每个`Transform`串连起来，第一个`Transform`接收来自javac编译的结果，以及已经拉取到在本地的第三方依赖（jar. aar），**还有resource资源，注意，这里的resource并非android项目中的res资源，而是asset目录下的资源。** 这些编译的中间产物，在`Transform`组成的链条上流动，每个`Transform`节点可以对class进行处理再传递给下一个`Transform`。我们常见的混淆，Desugar等逻辑，它们的实现如今都是封装在一个个`Transform`中，而我们自定义的`Transform`，会插入到这个`Transform`链条的最前面。

> 上面这幅图，只是展示Transform的其中一种情况。而Transform其实可以有两种输入，一种是消费型的，当前Transform需要将消费型型输出给下一个Transform，另一种是引用型的，当前Transform可以读取这些输入，而不需要输出给下一个Transform，比如Instant Run就是通过这种方式，检查两次编译之间的diff的。

最终，定义的Transform会被转化成一个个TransformTask，在Gradle编译时依次调用。

```
> com.android.build.gradle.internal.pipeline.TransformManager

/**
 * Adds a Transform.
 *
 * <p>This makes the current transform consumes whatever Streams are currently available and
 * creates new ones for the transform output.
 *
 * <p>his also creates a {@link TransformTask} to run the transform and wire it up with the
 * dependencies of the consumed streams.
 *
 * @param taskFactory the task factory
 * @param componentProperties the current scope
 * @param transform the transform to add
 * @param <T> the type of the transform
 * @return {@code Optional<AndroidTask<TaskProvider<TransformTask>>>} containing the AndroidTask
 *     for the given transform task if it was able to create it
 */
@NonNull
public <T extends Transform> Optional<TaskProvider<TransformTask>> addTransform(
        @NonNull TaskFactory taskFactory,
        @NonNull ComponentPropertiesImpl componentProperties,
        @NonNull T transform,
        @Nullable PreConfigAction preConfigAction,
        @Nullable TaskConfigAction<TransformTask> configAction,
        @Nullable TaskProviderCallback<TransformTask> providerCallback) {

    if (!validateTransform(transform)) {
        // validate either throws an exception, or records the problem during sync
        // so it's safe to just return null here.
        return Optional.empty();
    }

    if (!transform.applyToVariant(new VariantInfoImpl(componentProperties))) {
        return Optional.empty();
    }

    List<TransformStream> inputStreams = Lists.newArrayList();
    String taskName = componentProperties.computeTaskName(getTaskNamePrefix(transform));

    // get referenced-only streams
    List<TransformStream> referencedStreams = grabReferencedStreams(transform);

    // find input streams, and compute output streams for the transform.
    IntermediateStream outputStream =
            findTransformStreams(
                    transform,
                    componentProperties,
                    inputStreams,
                    taskName,
                    componentProperties.getGlobalScope().getBuildDir());

    if (inputStreams.isEmpty() && referencedStreams.isEmpty()) {
        // didn't find any match. Means there is a broken order somewhere in the streams.
        issueReporter.reportError(
                Type.GENERIC,
                String.format(
                        "Unable to add Transform '%s' on variant '%s': requested streams not available: %s+%s / %s",
                        transform.getName(),
                        componentProperties.getName(),
                        transform.getScopes(),
                        transform.getReferencedScopes(),
                        transform.getInputTypes()));
        return Optional.empty();
    }

    //noinspection PointlessBooleanExpression
    if (DEBUG && logger.isEnabled(LogLevel.DEBUG)) {
        logger.debug("ADDED TRANSFORM(" + componentProperties.getName() + "):");
        logger.debug("\tName: " + transform.getName());
        logger.debug("\tTask: " + taskName);
        for (TransformStream sd : inputStreams) {
            logger.debug("\tInputStream: " + sd);
        }
        for (TransformStream sd : referencedStreams) {
            logger.debug("\tRef'edStream: " + sd);
        }
        if (outputStream != null) {
            logger.debug("\tOutputStream: " + outputStream);
        }
    }

    transforms.add(transform);
    TaskConfigAction<TransformTask> wrappedConfigAction =
            t -> {
                t.getEnableGradleWorkers()
                        .set(
                                componentProperties
                                        .getGlobalScope()
                                        .getProjectOptions()
                                        .get(BooleanOption.ENABLE_GRADLE_WORKERS));
                if (configAction != null) {
                    configAction.configure(t);
                }
            };
    // create the task...
    return Optional.of(
            taskFactory.register(
                    new TransformTask.CreationAction<>(
                            componentProperties.getName(),
                            taskName,
                            transform,
                            inputStreams,
                            referencedStreams,
                            outputStream,
                            recorder),
                    preConfigAction,
                    wrappedConfigAction,
                    providerCallback));
}
```

## 2. 自定义Transform

自定义Transform首先需要先自定义一个插件，然后注册Transform

```
def isApp = target.plugins.hasPlugin(AppPlugin)
if (isApp) {
    println("apply to " + target.name)
    def appExtension = target.extensions.getByType(AppExtension)
    appExtension.registerTransform(new RegisterTransform(target))
}
```

自定义的Transform需要继承com.android.build.api.transform.Transform, 如下：

```
package com.example.plugin

import com.android.build.api.transform.*
import com.android.build.gradle.internal.pipeline.TransformManager
import org.apache.commons.io.FileUtils
import org.gradle.api.Project

class RegisterTransform extends Transform {
    Project project;

    RegisterTransform(Project project) {
        this.project = project;
    }

    @Override
    String getName() {
        return "RegisterTransform";
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    @Override
    boolean isIncremental() {
        return false;
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation);
        Collection<TransformInput> inputs = transformInvocation.getInputs()

        inputs.each { TransformInput input ->
            if (!input.jarInputs.empty) {
                input.jarInputs.each { JarInput jarInput ->
                    println("scan jar => ${jarInput.file.absolutePath}")
                    File destFile = transformInvocation.outputProvider.getContentLocation(
                            jarInput.name, jarInput.contentTypes, jarInput.scopes, Format.JAR)
                    FileUtils.copyFile(jarInput.file, destFile)
                }
            }

            if (!input.directoryInputs.empty) {
                input.directoryInputs.each { DirectoryInput directoryInput ->
                    File dest = transformInvocation.outputProvider.getContentLocation(
                            directoryInput.name, directoryInput.contentTypes, directoryInput.scopes, Format.DIRECTORY)
                    println("scan directory => ${dest.absolutePath}")
                    FileUtils.copyDirectory(directoryInput.file, dest)
                }
            }
        }
    }
}

```

每一个`Transform`都声明它的作用域, 作用对象以及具体的操作以及操作后输出的内容.

### **2.1 Name**

```
@Override
String getName() {
    return "RegisterTransform";
}
```

对应生成Transform task的名字

### **2.2 作用域**

通过`Transform#getScopes`方法我们可以声明自定义的transform的作用域, 指定作用域包括如下几种:

```
@Override
Set<? super QualifiedContent.Scope> getScopes() {
    return TransformManager.SCOPE_FULL_PROJECT;
}

/**
 * The scope of the content.
 *
 * <p>
 * This indicates what the content represents, so that Transforms can apply to only part(s)
 * of the classes or resources that the build manipulates.
 */
enum Scope implements ScopeType {
    /** Only the project (module) content */
    PROJECT(0x01),
    /** Only the sub-projects (other modules) */
    SUB_PROJECTS(0x04),
    /** Only the external libraries */
    EXTERNAL_LIBRARIES(0x10),
    /** Code that is being tested by the current variant, including dependencies */
    TESTED_CODE(0x20),
    /** Local or remote dependencies that are provided-only */
    PROVIDED_ONLY(0x40),

    /**
     * Only the project's local dependencies (local jars)
     *
     * @deprecated local dependencies are now processed as {@link #EXTERNAL_LIBRARIES}
     */
    @Deprecated
    PROJECT_LOCAL_DEPS(0x02),
    /**
     * Only the sub-projects's local dependencies (local jars).
     *
     * @deprecated local dependencies are now processed as {@link #EXTERNAL_LIBRARIES}
     */
    @Deprecated
    SUB_PROJECTS_LOCAL_DEPS(0x08);

    private final int value;

    Scope(int value) {
        this.value = value;
    }

    @Override
    public int getValue() {
        return value;
    }
}
```

![](<../../.gitbook/assets/image (95).png>)

| QualifiedContent.Scope |                     |
| ---------------------- | ------------------- |
| EXTERNAL\_LIBRARIES    | 只包含外部库              |
| PROJECT                | 只作用于project本身内容     |
| PROVIDED\_ONLY         | 支持compileOnly的远程依赖  |
| SUB\_PROJECTS          | 子模块内容               |
| TESTED\_CODE           | 当前变体测试的代码以及包括测试的依赖项 |

`TransformManager`整合了部分常用的Scope以及Content集合, 如果是`application`注册的transform, 通常情况下, 我们一般指定`TransformManager.SCOPE_FULL_PROJECT`;如果是`library`注册的transform, 我们只能指定`TransformManager.PROJECT_ONLY` , 我们可以在`LibraryTaskManager#createTasksForVariantScope`中看到相关的限制报错代码

### **2.3 作用对象**

通过`Transform#getInputTypes`我们可以声明其的作用对象, 我们可以指定的作用对象只包括两种

![](<../../.gitbook/assets/image (33).png>)

| QualifiedContent.ContentType |                                    |
| ---------------------------- | ---------------------------------- |
| CLASSES                      | Java代码编译后的内容, 包括文件夹以及Jar包内的编译后的类文件 |
| RESOURCES                    | 基于资源获取到的内容                         |

### **2.4 增量编译**

当我们开启增量编译的时候，相当input包含了changed/removed/added三种状态，实际上还有notchanged。需要做的操作如下：

* NOTCHANGED: 当前文件不需处理，甚至复制操作都不用；
* ADDED、CHANGED: 正常处理，输出给下一个任务；
* REMOVED: 移除outputProvider获取路径对应的文件。

## 3. Transform编写模板

基本的拷贝实现（不实现会无法编译apk），不带增量编译

```
class CustomTransform extends Transform {
    Project project;


    RegisterTransform(Project project) {
        this.project = project;
    }


    final String NAME = "CustomTransform"

    @Override
    String getName() {
        return NAME
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)

        //OutputProvider管理输出路径，如果消费型输入为空，你会发现OutputProvider == null
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();

        transformInvocation.inputs.each { TransformInput input ->
            input.jarInputs.each { JarInput jarInput ->
                //处理Jar
                processJarInput(jarInput, outputProvider)
            }

            input.directoryInputs.each { DirectoryInput directoryInput ->
                //处理源码文件
                processDirectoryInputs(directoryInput, outputProvider)
            }
        }
    }

    void processJarInput(JarInput jarInput, TransformOutputProvider outputProvider) {
        File dest = outputProvider.getContentLocation(
                jarInput.getFile().getAbsolutePath(),
                jarInput.getContentTypes(),
                jarInput.getScopes(),
                Format.JAR)
        //TODO do some transform
        //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
        FileUtils.copyFile(jarInput.getFile(), dest)
    }

    void processDirectoryInputs(DirectoryInput directoryInput, TransformOutputProvider outputProvider) {
        File dest = outputProvider.getContentLocation(directoryInput.getName(),
                directoryInput.getContentTypes(), directoryInput.getScopes(),
                Format.DIRECTORY)
        //建立文件夹
        FileUtils.forceMkdir(dest)
        //TODO do some transform
        //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
        FileUtils.copyDirectory(directoryInput.getFile(), dest)
    }
}
```

**带增量编译：**

```
class CustomTransform extends Transform {
    Project project;

    RegisterTransform(Project project) {
        this.project = project;
    }

    final String NAME = "CustomTransform"

    @Override
    String getName() {
        return NAME
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return true
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)

        boolean isIncremental = transformInvocation.isIncremental()

        //OutputProvider管理输出路径，如果消费型输入为空，你会发现OutputProvider == null
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider()

        if (!isIncremental) {
            //不需要增量编译，先清除全部
            outputProvider.deleteAll()
        }

        transformInvocation.getInputs().each { TransformInput input ->
            input.jarInputs.each { JarInput jarInput ->
                //处理Jar
                processJarInputWithIncremental(jarInput, outputProvider, isIncremental)
            }

            input.directoryInputs.each { DirectoryInput directoryInput ->
                //处理文件
                processDirectoryInputWithIncremental(directoryInput, outputProvider, isIncremental)
            }
        }
    }

    void processJarInputWithIncremental(JarInput jarInput, TransformOutputProvider outputProvider, boolean isIncremental) {
        File dest = outputProvider.getContentLocation(
                jarInput.getFile().getAbsolutePath(),
                jarInput.getContentTypes(),
                jarInput.getScopes(),
                Format.JAR)
        if (isIncremental) {
            //处理增量编译
            processJarInputWhenIncremental(jarInput, dest)
        } else {
            //不处理增量编译
            processJarInput(jarInput, dest)
        }
    }

    void processJarInput(JarInput jarInput, File dest) {
        transformJarInput(jarInput, dest)
    }

    void processJarInputWhenIncremental(JarInput jarInput, File dest) {
        switch (jarInput.status) {
            case Status.NOTCHANGED:
                break
            case Status.ADDED:
            case Status.CHANGED:
                //处理有变化的
                transformJarInputWhenIncremental(jarInput.getFile(), dest, jarInput.status)
                break
            case Status.REMOVED:
                //移除Removed
                if (dest.exists()) {
                    FileUtils.forceDelete(dest)
                }
                break
        }
    }

    void transformJarInputWhenIncremental(JarInput jarInput, File dest, Status status) {
        if (status == Status.CHANGED) {
            //Changed的状态需要先删除之前的
            if (dest.exists()) {
                FileUtils.forceDelete(dest)
            }
        }
        //真正transform的地方
        transformJarInput(jarInput, dest)
    }

    void transformJarInput(JarInput jarInput, File dest) {
        //TODO do some transform
        //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
        FileUtils.copyFile(jarInput.getFile(), dest)
    }

    void processDirectoryInputWithIncremental(DirectoryInput directoryInput, TransformOutputProvider outputProvider, boolean isIncremental) {
        File dest = outputProvider.getContentLocation(
                directoryInput.getFile().getAbsolutePath(),
                directoryInput.getContentTypes(),
                directoryInput.getScopes(),
                Format.DIRECTORY)
        if (isIncremental) {
            //处理增量编译
            processDirectoryInputWhenIncremental(directoryInput, dest)
        } else {
            processDirectoryInput(directoryInput, dest)
        }
    }

    void processDirectoryInputWhenIncremental(DirectoryInput directoryInput, File dest) {
        FileUtils.forceMkdir(dest)
        String srcDirPath = directoryInput.getFile().getAbsolutePath()
        String destDirPath = dest.getAbsolutePath()
        Map<File, Status> fileStatusMap = directoryInput.getChangedFiles()
        fileStatusMap.each { Map.Entry<File, Status> entry ->
            File inputFile = entry.getKey()
            Status status = entry.getValue()
            String destFilePath = inputFile.getAbsolutePath().replace(srcDirPath, destDirPath)
            File destFile = new File(destFilePath)
            switch (status) {
                case Status.NOTCHANGED:
                    break
                case Status.REMOVED:
                    if (destFile.exists()) {
                        FileUtils.forceDelete(destFile)
                    }
                    break
                case Status.ADDED:
                case Status.CHANGED:
                    FileUtils.touch(destFile)
                    transformSingleFile(inputFile, destFile, srcDirPath)
                    break
            }
        }
    }

    void processDirectoryInput(DirectoryInput directoryInput, File dest) {
        transformDirectoryInput(directoryInput, dest)
    }

    void transformDirectoryInput(DirectoryInput directoryInput, File dest) {
        //TODO do some transform
        //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
        FileUtils.copyDirectory(directoryInput.getFile(), dest)
    }

    void transformSingleFile(File inputFile, File destFile, String srcDirPath) {
        FileUtils.copyFile(inputFile, destFile)
    }
}

```

## 4. Transform优化

一般就三种：

* 增量编译
* 并发编译
* include... exclude...缩小transform范围

并发编译，简单实现如下：

```
  WaitableExecutor waitableExecutor = WaitableExecutor.useGlobalSharedThreadPool()
  
  ......
  transformInvocation.getInputs().each { TransformInput input ->
            input.jarInputs.each { JarInput jarInput ->
                //多线程处理Jar
                waitableExecutor.execute(new Callable<Object>() {
                    @Override
                    Object call() throws Exception {
                        processJarInputWithIncremental(jarInput, outputProvider, isIncremental)
                        return null
                    }
                })
            }

            input.directoryInputs.each { DirectoryInput directoryInput ->
                //多线程处理文件
                waitableExecutor.execute(new Callable<Object>() {
                    @Override
                    Object call() throws Exception {
                        processDirectoryInputWithIncremental(directoryInput, outputProvider, isIncremental)
                        return null
                    }
                })
            }
        }

        //等待所有任务结束
        waitableExecutor.waitForTasksWithQuickFail(true)
```

## 4. 参考

* [Android Gradle plugin API reference](https://developer.android.com/reference/tools/gradle-api)
* [Transform详解](https://www.jianshu.com/p/37a5e058830a)
