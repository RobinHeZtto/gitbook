# 插件化

## 1. 什么是插件化？

插件化技术是指在不安装apk的情况下，支持插件apk的加载与执行。插件化技术最初源于免安装运行apk的想法，这个免安装的apk就可以理解为插件，而支持插件的 app 我们一般叫宿主。

### 1.1 为什么需要插件化？

* APP的功能模块越来越多，体积越来越大
* 模块之间的耦合度高，协同开发沟通成本越来越大
* 方法数目可能超过65535，APP占用的内存过大
* 应用之间的互相调用

### 1.2 插件化与组件化的区别

组件化开发就是将一个app分成多个模块，每个模块都是一个组件，开发的过程中我们可以让这些组件相互依赖或者单独调试部分组件等，但是最终发布的时候是将这些组件合并统一成一个apk，这就是组件化开发。

插件化开发和组件化略有不同，插件化开发是将整个app拆分成多个模块，这些模块包括一个宿主和多个插件，每个模块都是一个apk，最终打包的时候宿主apk和插件apk分开打包。

### 1.3 各大插件化框架对比

![](<../../../.gitbook/assets/image (203).png>)

## 2. 插件化实现原理

要想实现插件化，需要考虑以下三个问题：

1. 如何加载插件的类？
2. 如何加载插件的资源？
3. 如何调用插件类？

第一个问题，如何加载插件的类？我们知道在Java/Android中类都是通过ClassLoader来加载的，那么首先来看一下Android中的ClassLoader。

### 2.1 如何加载插件的类？

#### Android ClassLoader

在Android中，Classloader的层次结构如下所示：

![](<../../../.gitbook/assets/image (345).png>)

如上图，在Android中有BootClassloader，PathClassloader，DexClassloader。其中BootClassloader是用来加载系统Framework核心类，PathClassloader用来加载我们App中的类。如下代码输出的结果，activity等Framework中的类是由BootClassloader加载的，MainActivity等打包在应用中的类是由PathClassloader加载的。

```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        Log.d("Test", "activity classloader ${Activity::class.java.classLoader}")
        Log.d("Test", "MainActivity classloader ${MainActivity::class.java.classLoader}")

    }
}

// 测试结果
Test: activity classloader java.lang.BootClassLoader@4300d48
Test: MainActivity classloader dalvik.system.PathClassLoader[DexPathList
[[zip file "/data/app/com.he.ztto.mydroidplugin-ScCWJgAb1hzyQxnkPo_Dvg
==/base.apk"],nativeLibraryDirectories=[/data/app/com.he.ztto.mydroidplugin
-ScCWJgAb1hzyQxnkPo_Dvg==/lib/x86, /system/lib, /system/product/lib]]]
```

DexClassLoader与PathClassloader的区别？

* DexClassLoader：能够加载未安装的jar/apk/dex&#x20;
* PathClassLoader：只能加载系统中已经安装过的apk

注意⚠️：网上的都是这种说法，在8.0（API 26）之前，它们二者的唯一区别是第二个参数 optimizedDirectory，这个参数的意思是生成的 odex（优化的dex）存放的路径。**在8.0（API 26）及之后，二者就完全一样了。**

[DexClassLoader](https://cs.android.com/android/platform/superproject/+/master:libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java?hl=zh-cn):

```
libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java

public class DexClassLoader extends BaseDexClassLoader {

    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```

[PathClassLoader](https://cs.android.com/android/platform/superproject/+/master:libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java?hl=zh-cn):

```
public class PathClassLoader extends BaseDexClassLoader {

    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }

    public PathClassLoader(
            String dexPath, String librarySearchPath, ClassLoader parent,
            ClassLoader[] sharedLibraryLoaders) {
        super(dexPath, librarySearchPath, parent, sharedLibraryLoaders);
    }
}

```

#### 类加载机制

双亲委派机制：

当类加载器收到类加载请求时，首先检测这个类是否已经被加载了，如果已经加载了，直接获取并返回。如果没有被加载，parent 不为 null，则调用parent的loadClass进行加载，依次递归，如果找到了或者加载了就返回，如果即没找到也加载不了，才自己去加载。



![](<../../../.gitbook/assets/image (220).png>)



```
libcore/ojluni/src/main/java/java/lang/ClassLoader.java

protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                c = findClass(name);
            }
        }
        return c;
}

```





## 参考资料

*   [Android类加载器ClassLoader](http://gityuan.com/2017/03/19/android-classloader/)



