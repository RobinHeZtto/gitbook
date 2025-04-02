# APT详解

## 1. APT是什么？

APT(Annotation Process Tool)是处理注释的工具，它对源代码文件进行检测找出其中的Annotation，并对其进行自定义处理。

## 2. 如何自定义APT

注解处理器需要单独创建一个 Java/Kotlin Library 子模块来存放，然后继承并实现 AbstractProcessor 抽象类。

AbstractProcessor的定义如下:  [AbstractProcessor](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/AbstractProcessor.html)&#x20;

![](<../../.gitbook/assets/image (208).png>)

**自定义APT**

| 变量和类型                                                                                                                                                                                                      | 方法                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 描述                                                                                                                                                                                                      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`Iterable`](https://www.apiref.com/java11-zh/java.base/java/lang/Iterable.html)`<? extends` [`Completion`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/Completion.html)`>` | [`getCompletions`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#getCompletions\(javax.lang.model.element.Element,javax.lang.model.element.AnnotationMirror,javax.lang.model.element.ExecutableElement,java.lang.String\))`​(`[`Element`](https://www.apiref.com/java11-zh/java.compiler/javax/lang/model/element/Element.html) `element,` [`AnnotationMirror`](https://www.apiref.com/java11-zh/java.compiler/javax/lang/model/element/AnnotationMirror.html) `annotation,` [`ExecutableElement`](https://www.apiref.com/java11-zh/java.compiler/javax/lang/model/element/ExecutableElement.html) `member,` [`String`](https://www.apiref.com/java11-zh/java.base/java/lang/String.html) `userText)` | 返回一个空的迭代完成。                                                                                                                                                                                             |
| [`Set`](https://www.apiref.com/java11-zh/java.base/java/util/Set.html)`<`[`String`](https://www.apiref.com/java11-zh/java.base/java/lang/String.html)`>`                                                   | [`getSupportedAnnotationTypes`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#getSupportedAnnotationTypes\(\))`()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | 如果处理器类使用[`SupportedAnnotationTypes`进行](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/SupportedAnnotationTypes.html)批注，则返回与注释具有相同字符串集的不可修改集。                                |
| [`Set`](https://www.apiref.com/java11-zh/java.base/java/util/Set.html)`<`[`String`](https://www.apiref.com/java11-zh/java.base/java/lang/String.html)`>`                                                   | [`getSupportedOptions`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#getSupportedOptions\(\))`()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | 如果处理器类使用[`SupportedOptions`进行](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/SupportedOptions.html)批注，则返回具有与批注相同的字符串集的不可修改集。                                               |
| [`SourceVersion`](https://www.apiref.com/java11-zh/java.compiler/javax/lang/model/SourceVersion.html)                                                                                                      | [`getSupportedSourceVersion`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#getSupportedSourceVersion\(\))`()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 如果处理器类使用[`SupportedSourceVersion`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/SupportedSourceVersion.html)进行批注，请在批注中返回源版本。                                              |
| `void`                                                                                                                                                                                                     | [`init`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#init\(javax.annotation.processing.ProcessingEnvironment\))`​(`[`ProcessingEnvironment`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/ProcessingEnvironment.html) `processingEnv)`                                                                                                                                                                                                                                                                                                                                                                                                                                | 通过将 `processingEnv`字段设置为 `processingEnv`参数的值，使用处理环境初始化处理器。                                                                                                                                              |
| `protected boolean`                                                                                                                                                                                        | [`isInitialized`](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#isInitialized\(\))`()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | 返回 `true`如果该对象已 [initialized](https://www.apiref.com/java11-zh/java.compiler/javax/annotation/processing/AbstractProcessor.html#init\(javax.annotation.processing.ProcessingEnvironment\)) ， `false`否则。 |

AbstractProcessor的定义:  [AbstractProcessor](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/AbstractProcessor.html)&#x20;



一般我们需重写的方法：

* `init(ProcessingEnvironment env)`: 每一个注解处理器类都必须有一个空的构造函数。然而，这里有一个特殊的`init()`方法，它会被注解处理工具调用，并输入`ProcessingEnviroment`参数。`ProcessingEnviroment`提供很多有用的工具类`Elements`, `Types`和`Filer`。
* `process(Set<? extends TypeElement> annotations, RoundEnvironment env)`: 这相当于每个处理器的主函数`main()`。可以在这里写扫描、评估和处理注解的代码，以及生成Java文件。输入参数`RoundEnviroment`，可以查询出包含特定注解的被注解元素。
* `getSupportedAnnotationTypes()`: 这里你必须指定，这个注解处理器是注册给哪个注解的。注意，它的返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。
* `getSupportedSourceVersion()`: 用来指定使用的Java版本。通常这里返回`SourceVersion.latestSupported()`。

**如何给注解处理器传参：**

可以通过在build.gradle中指定**annotationProcessorOptions**，如下：

```
android {
    defaultConfig {
        javaCompileOptions 
            annotationProcessorOptions {
                argument "key1", "value1"
                argument "key2", "value2"
            }
        }
    }
 }
```

在 Processor 的 init 方法中可以获取参数：

```
@Override
public synchronized void init(ProcessingEnvironment env) {
    super.init(env);

    String value1 = env.getOptions().get("key1");
}
```

**注册Processor**

1.手动注册

在main/下创建`resources/META-INF/services` 目录，然后创建名为javax.annotation.processing.Processor的文件。

![](<../../.gitbook/assets/image (94).png>)

文件中写上自定义注解处理器的类名，如下：

![](<../../.gitbook/assets/image (30).png>)

2\. 自动注册

在build.gradle中加入auto-service依赖。

```
implementation "com.google.auto.service:auto-service:1.0-rc7"
```

然后在自定义注解处理器上加上@AutoService注解即可。`AutoService`注解处理器是Google开发的，用来生成`META-INF/services/javax.annotation.processing.Processor`文件的。

```
@AutoService(Processor::class)
class PerfProcessor : AbstractProcessor(){
   ......
}
```

注意⚠️：如果使用kotlin，需要做如下配置，才可自动注册成功。

```
plugins {
    id 'kotlin-kapt'
}

dependencies {
    kapt("com.google.auto.service:auto-service:1.0-rc7")
    implementation "com.google.auto.service:auto-service:1.0-rc7"
}
```

javac会自动检查和读取`javax.annotation.processing.Processor`中的内容，并且注册文件指定的class作为注解处理器。\
\
总结如下：

* 注解处理过程是一个有序的循环过程。在每次循环中，一个处理器可能被要求去处理那些在上一次循环中产生的源文件和类文件中的注解。Annotation Processor被调用一次后，后续若还有注解处理,该Annotation Processor仍然会继续被调用。
* 自定义Annotation Processor必须带有一个无参构造函数，让javac进行实例化。
* 如果Annotation Processor抛出一个未捕获异常,javac可能会停止其他的Annotation Processor.只有在无法抛出错误或报告的情况下,才允许抛出异常.
* Annotation Processor运行在一个独立的jvm中,所以可以将它看成是一个java应用程序.

## **4. Element**

**Element**是APT技术的基础，代表程序的一个元素，这个元素可以是：包、类/接口、属性变量、方法/方法形参、泛型参数。\
\
Element代表的是源代码，TypeElement代表的是源代码中的类型元素，例如类。但是TypeElement并不包含类本身的信息，可以从TypeElement中获取类的名字，但是获取不到类的信息，例如它的父类这种信息需要通过`TypeMirror`获取。可以通过调用`elements.asType()`获取元素的`TypeMirror`

判断是否是作用在class上的注解:

```
element.getKind().isClass()
```

判断是否是某个class的子类：

```
private boolean isSubtype(Element typeElement, String type) {
    return processingEnv.getTypeUtils().isSubtype(typeElement.asType(),
            processingEnv.getElementUtils().getTypeElement(type).asType());
}
```

判断修饰符：

```
 Set<Modifier> modifiers = typeElement.getModifiers();
 modifiers.contains(Modifier.ABSTRACT) ???
```

## 3. APT的使用场景

动态生成代码，代码检查，等等....
