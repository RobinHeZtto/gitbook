# 包体积优化

![](<../../.gitbook/assets/image (17).png>)

## 1. APK 结构

APK文件本质上是一个**Zip** 压缩文件，其中包含构成应用的所有文件。这些文件包括编译的代码、资源、so库等。

![](<../../.gitbook/assets/image (231).png>)

| 文件/目录               | 描述                                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------------- |
| lib/                | 存放so文件，可能会有armeabi、armeabi-v7a、arm64-v8a、x86、x86\_64、mips，大多数情况下只需要支持armabi与x86的架构即可，如果非必需，可以考虑拿掉x86的部分 |
| res/                | 存放编译后的资源文件，例如：drawable、layout等等                                                                         |
| assets/             | 应用程序的资源，应用程序可以使用AssetManager来检索该资源                                                                      |
| META-INF/           | 该文件夹一般存放于已经签名的APK中，它包含了APK中所有文件的签名摘要等信息                                                                 |
| classes(n).dex      | classes文件是Java Class，被DEX编译后可供Dalvik/ART虚拟机所理解的文件格式                                                     |
| resources.arsc      | 编译后的二进制资源文件                                                                                             |
| AndroidManifest.xml | Android的清单文件，格式为AXML，用于描述应用程序的名称、版本、所需权限、注册的四大组件                                                        |

## 2. 优化方法

使用Androd Studio自带的APK apkanalyzer对比二个APK

### 代码优化

* 统一三方库的使用
* 模块拆分，根据需求编译打包需要的模块
*   代码缩减（即摇树优化）minifyEnabled true

    一方面可以从应用及其库依赖项中检测并安全地移除不使用的类、字段、方法和属性，另外一方面可以通过混淆缩短类和成员的名称，从而减小 DEX 文件的大小。

### 资源优化

* 资源缩减\
  启用shrinkResources ，则 Gradle 在打包APK时可以自动忽略未使用资源。 资源缩减只有在与代码缩减：minifyEnabled 配合使用时才能发挥作用。在代码缩减器移除所有不 使用的代码后，资源缩减器便可确定应用仍要使用的资源 。

```
android {
    ...
    buildTypes {
        release {
            shrinkResources true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
}
```

* 移除未使用的资源\
  lint 工具是 Android Studio 中附带的静态代码分析器，可以帮助我们检测代码中未使用的资源。

![](<../../.gitbook/assets/image (336).png>)

![](<../../.gitbook/assets/image (79).png>)

![](<../../.gitbook/assets/image (62).png>)

![](<../../.gitbook/assets/image (320).png>)

如果有想要特别声明需要保留或舍弃的特定资源，在项目中创建一个包含 标记的 XML 文件，并 在 tools:keep 属性中指定每个要保留的资源，在 **tools:discard** 属性中指定每个要舍弃的资源。这两个属性都 接受以逗号分隔的资源名称列表。还可以将星号字符用作通配符。 \<?xml version="1.0" encoding="utf-8"?>  将该文件保存在项目资源中，例如，保存在 **res/raw/keep.xml** 中。构建系统不会将此文件打包到 APK 中。

* 一键移除没有使用的资源

![](<../../.gitbook/assets/image (212).png>)

* 动态库打包配置\
  so文件是由ndk编译出来的动态库，是 c/c++ 写的，所以不是跨平台的。ABI 是应用程序二进制接口简称 （Application Binary Interface），定义了二进制文件（尤其是.so文件）如何运行在相应的系统平台上，从使用的指令集，内存对齐到可用的系统函数库。\
  在Android 系统中，每一个CPU架构对应一个ABI，目前支持的有： armeabi-v7a，arm64- v8a，x86，x86\_64。目前市面上手机设备基本上都是arm架构， armeabi-v7a 几乎能兼容 所有设备。因此可以配置：&#x20;

```
android{ 
    defaultConfig{ 
        ndk{
            abiFilters "armeabi-v7a" 
        } 
    } 
}
```

* 图片资源，尽量不使用png，将PNG，JPG转成webp，小图标使用矢量图。

![](<../../.gitbook/assets/image (352).png>)

![](<../../.gitbook/assets/image (246).png>)

* [AndResGuard](https://github.com/shwenzhang/AndResGuard) 将原本冗长的资源路径变短，例如将`res/drawable/wechat`变为`r/d/a`

### 其他

* Dex debugItem  [https://juejin.im/post/6844903712201277448](https://juejin.im/post/6844903712201277448)
* app bundle [https://developer.android.google.cn/guide/app-bundle](https://developer.android.google.cn/guide/app-bundle)

## 3. 参考资料

* [Android官方 使用 APK 分析器分析您的构建](https://developer.android.com/studio/build/apk-analyzer?hl=zh-cn)
* [Android官方 缩减应用大小](https://developer.android.com/topic/performance/reduce-apk-size)
* [Android官方 使用 lint 检查改进您的代码](https://developer.android.com/studio/write/lint)
* [抖音 包大小优化：资源优化](https://www.infoq.cn/article/itvefvgd5uv6r07nhssx)
* [美团 Android App包瘦身优化实践](https://tech.meituan.com/2017/04/07/android-shrink-overall-solution.html)
