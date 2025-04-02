# 组件化-路由

## 1. **为什么需要路由框架**

### 原生路由

原生路由方案一般是通过显式intent和隐式intent两种方式实现的，而在显式intent的情况下，因为会存在直接的类依赖的问题，导致耦合非常严重；而在隐式intent情况下，则会出现规则集中式管理，导致协作变得非常困难。而且一般而言配置规则都是在Manifest中的，这就导致了扩展性较差。除此之外，使用原生的路由方案会出现跳转过程无法控制的问题，因为一旦使用了StartActivity()就无法插手其中任何环节了，只能交给系统管理，这就导致了在跳转失败的情况下无法降级，而是会直接抛出运营级的异常。

![](<../../../.gitbook/assets/image (399).png>)

### **路由框架的特点**

1. **分发**：把一个URL或者请求按照一定的规则分配给一个服务或者页面来处理，这个流程就是分发，分发是路由框架最基本的功能，当然也可以理解成为简单的跳转。
2. **管理**：将组件和页面按照一定的规则管理起来，在分发的时候提供搜索、加载、修改等操作，这部分就是管理，也是路由框架的基础，上层功能都是建立在管理之上。
3. **控制**：就像路由器一样，路由的过程中，会有限速、屏蔽等一些控制操作，路由框架也需要在路由的过程中，对路由操作做一些定制性的扩展，比方刚才提到的AOP，后期的功能更新，也是围绕这个部分来做的。

![](<../../../.gitbook/assets/image (40).png>)

## 2. Arouter路由方案

![](<../../../.gitbook/assets/image (339).png>)

![](<../../../.gitbook/assets/image (209).png>)

### 核心数据结构

#### RouteMate：

![](<../../../.gitbook/assets/image (236).png>)

RouteMeta 包含了路由跳转的基本信息，其中RouteType为路由的类型，包括Activity跳转等。

```
public enum RouteType {
    ACTIVITY(0, "android.app.Activity"),
    SERVICE(1, "android.app.Service"),
    PROVIDER(2, "com.alibaba.android.arouter.facade.template.IProvider"),
    CONTENT_PROVIDER(-1, "android.app.ContentProvider"),
    BOARDCAST(-1, ""),
    METHOD(-1, ""),
    FRAGMENT(-1, "android.app.Fragment"),
    UNKNOWN(-1, "Unknown route type");

    ......
}
```

路由原始类型 rawType 的类型为 Element，Element 中包含了跳转目标的 Class 信息，是由路由处理器 RouteProcessor 设定的。

#### Postcard

![](<../../../.gitbook/assets/image (325).png>)

Postcard 继承于RouteMeta， 并且包含如下字段。

* 统一资源标识符 uri
* 标签 tag
* 传参 mBundle
* 标志 flags
* 超时时间 timeout
* 自定义服务 provider
* 是否走绿色通道 greenChannel
* 序列化服务 serializationService
* 转场动画 optionsCompat
* 进入与退出动画 enterAnim/exitAnim

### 路由生成机制

Arouter路由信息由注解处理器自动生成，

![](<../../../.gitbook/assets/image (18).png>)

RouteProcessor 对于声明了 @Route 注解的类的处理大致可分为下面 4 个步骤

1. 获取路由元素
2. 创建路由元信息
3. 把路由元信息进行分组
4. 生成路由文件

![](<../../../.gitbook/assets/image (54).png>)

**1. 获取路由元素**

这里的元素指的是 javax 包中的 Element ，Element 表示 Java 语言元素，比如字段、包、方法、类以及接口。

process() 方法会接收到 annotations 和 roundEnv 两个参数，annotations 就是当前处理器要处理的注解，roundEnv 就是运行环境 JavacRoundEnvironment。

JavacRoundEnvironment 有一个 getElementsAnnotatedWith() 方法，RouteProcessor 在处理注解时首先会用这个方法获取元素。

**2. 创建路由元信息**

![](<../../../.gitbook/assets/image (397).png>)

这里说的路由元信息指的是 RouteMeta，RouteProcessor 会把声明了 @Route 注解的的 Activity、Provider、Service 或 Fragment 和一个 RouteMeta 关联起来。

当元素类型为 Activity 时，RouteProcessor 会遍历获取 Activity 的子元素，也就是 Activity 的成员变量，把它们放入注入配置 RouteMeta 的 injectConfig 中，当我们配置了要生成路由文档时，RouteProcessor 就会把 injectConfig 写入到文档中，然后就对元信息进行分组。

**3. 把路由元信息进行分组**

在 RouteProcessor 中有一个 groupMap，在 RouteMeta 创建好后，RouteProcessor 会把不同的 RouteMeta 进行分组，放入到 groupMap 中。

拿路径 /goods/details 来说，如果我们在 @Route 中没有设置 group 的值，那么 RouteProcessor 就会把 goods 作为 RouteMeta 的 group 。

**4. 生成路由表**

![](<../../../.gitbook/assets/image (371).png>)

当 RouteProcessor 把 RouteMeta 分组好后，就会用 JavaPoet 生成 Group、Provider 和 Root 路由文件，路由表就是由这些文件组成的，JavaPoet 是 Square 开源的代码生成框架。

生成路由文件后，物流中心 LogisticsCenter 需要用这些文件来填充仓库 Warehouse 中的 routes 和 providerIndex 等索引，然后在跳转时根据 routes 和索引来跳转。

关于 LogisticsCenter 和 Warehouse 在后面会讲，下面我们来看下路由文件的内容。

RouteProcessor 生成的路由文件位于build/generated/source/kapt/(debug/release)/com/alibaba/android/arouter/routes。

![](<../../../.gitbook/assets/image (225).png>)

### 路由加载注册

路由表生成以后，还是散落在各处，在初始化时需要统一加载注册。

```
/**
 * arouter-auto-register plugin will generate code inside this method
 * call this method to register all Routers, Interceptors and Providers
 * @author billy.qi <a href="mailto:qiyilike@163.com">Contact me.</a>
 * @since 2017-12-06
 */
private static void loadRouterMap() {
    registerByPlugin = false;
    //auto generate register code by gradle plugin: arouter-auto-register
    // looks like below:
    // registerRouteRoot(new ARouter..Root..modulejava());
    // registerRouteRoot(new ARouter..Root..modulekotlin());
}
```

Arouter路由表支持二种加载注册的方式：

**一种是编译时通过字节码注入方式自动注册，另外一种是从dex中加载路由表注册；**

在我们调用 ARouter 的 init() 方法时，ARouter 会调用 LogisticsCenter 的 init() 方法，在 LogisticsCenter 的 init() 方法中，会判断当前路由表加载方式是否为插件，不是的话则从 Dex 中加载路由表，是的话则由插件从 Jar 中加载路由表。

### 路由查找跳转

当我们调用 Postcard 的 navigation() 方法时，Postcard 会调用 \_ARouter 的 navigation() 方法，然后 \_ARouter 才会去加载路由表，所以我们来看下 navigation() 的处理流程。

\_ARouter 的 navigation() 方法有下面两种重载。

1. navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback)
2. navigation(Class\<? extends T> service)

![](<../../../.gitbook/assets/image (47).png>)

\_ARouter 的 navigation() 方法的大致处理流程如下：

1.  预处理服务

    navigation() 首先会根据我们实现的预处理服务，判断是否继续往下处理，不往下处理则中断跳转流程；
2.  完善明信片

    如果预处理服务返回的是 true ，那么 navigation() 方法就会加载路由表，把 RouteMeta 的信息填充到 Postcard 中，比如终点 destination 等信息。这个操作是由物流中心 LogisticsCenter 做的。
3.  降级策略

    假如 LogisticsCenter 在完善明信片的过程中遇到了异常，比如找不到路径对应的目标，那么就会调用降级策略，我们可以在降级策略中显示错误提示等信息；
4.  拦截器链

    在明信片完善信息后，navigation() 就会把跳转事件交给拦截器链处理；
5.  按类型跳转

    在拦截器链处理完成，并且没有中断跳转时，navigation() 就会按照路径类型跳转到不同的页面或调用自定义服务；

#### 预处理服务

**1. 实现预处理服务**

预处理服务可以让我们在 ARouter 进行跳转前，根据 PostCard 的内容判断是否要独立地对这次跳转进行处理，是的话则在 onPretreatment() 中返回 false 即可

![](<../../../.gitbook/assets/image (19).png>)

在 navigation() 中，首先会调用预处理服务的 onPretreamtn() 方法，判断是否要继续往下处理，如果返回结果为 false ，则不再往下处理，也就是不会进行跳转等操作。

```
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
    if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
        // Pretreatment failed, navigation canceled.
        return null;
    }
    // ......
}    
```

**2. 完善PostCard**

调用完预处理服务后，\_ARouter 就会用物流中心 LogisticsCenter 来加载路由表，路由表也就是 RouteProcessor 生成的路由文件。

```
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
    if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
        // Pretreatment failed, navigation canceled.
        return null;
    }

    try {
        LogisticsCenter.completion(postcard);
    } catch (NoRouteFoundException ex) {
        // ......
    }

    ......
    return null;
}
```

整个过程如下所示：

![](<../../../.gitbook/assets/image (319).png>)

**获取路由元信息：**

![](<../../../.gitbook/assets/image (391).png>)

在 \_ARouter 初始化时，会把 LogisticsCenter 也进行初始化，而 LogisticsCenter 的初始化方法中，会读取 RouteProcessor 创建好的路由表，然后放到对应的索引 index 中。有了索引，当 \_ARouter 调用 LogisticsCenter 的 completion() 方法时，就可以用索引从 Warehouse 的 routes 中获取路由元信息。

如果 LogisticsCenter 根据索引查找不到对应的 RouteMeta，那就说明 routes 还没有被填充，这时 LogisticsCenter 就会获取 group 的 RouteMeta，然后把 group 下的路径填充到 routes 中，然后再调用一次 completion() ，这时就可以取填充明信片的信息了。

```
public synchronized static void completion(Postcard postcard) {
    if (null == postcard) {
        throw new NoRouteFoundException(TAG + "No postcard!");
    }

    // 先直接从rotues里面拿
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    // 如果拿不到，说明没有加载过
    if (null == routeMeta) { 
        // 从groupsIndex里面拿所在的group  
        Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
        // group信息为空，说明没有路由信息，异常
        if (null == groupMeta) {
            throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
        } else {
            // Load route and cache it into memory, then delete from metas.
            try {
                if (ARouter.debuggable()) {
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                }
                // 拿到group路由信息，先实例化
                IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                // load整个group的路由信息到routes
                iGroupInstance.loadInto(Warehouse.routes);
                // group中的信息已加载，移除group记录
                Warehouse.groupsIndex.remove(postcard.getGroup());

                if (ARouter.debuggable()) {
                    logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                }
            } catch (Exception e) {
                throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
            }
            // 再次调用完善postcard信息，
            completion(postcard);   // Reload
        }
    } else {
        // 设置路由的Destination，这个是路由的核心
        postcard.setDestination(routeMeta.getDestination());
        postcard.setType(routeMeta.getType());
        postcard.setPriority(routeMeta.getPriority());
        postcard.setExtra(routeMeta.getExtra());

        Uri rawUri = postcard.getUri();
        if (null != rawUri) {   // Try to set params into bundle.
            Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
            Map<String, Integer> paramsType = routeMeta.getParamsType();

            if (MapUtils.isNotEmpty(paramsType)) {
                // Set value by its type, just for params which annotation by @Param
                for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                    setValue(postcard,
                            params.getValue(),
                            params.getKey(),
                            resultMap.get(params.getKey()));
                }

                // Save params name which need auto inject.
                postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
            }

            // Save raw uri
            postcard.withString(ARouter.RAW_URI, rawUri.toString());
        }

        switch (routeMeta.getType()) {
            case PROVIDER:  // if the route is provider, should find its instance
                // Its provider, so it must implement IProvider
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) { // There's no instance of this provider
                    IProvider provider;
                    try {
                        provider = providerMeta.getConstructor().newInstance();
                        provider.init(mContext);
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        throw new HandlerException("Init provider failed! " + e.getMessage());
                    }
                }
                postcard.setProvider(instance);
                postcard.greenChannel();    // Provider should skip all of interceptors
                break;
            case FRAGMENT:
                postcard.greenChannel();    // Fragment needn't interceptors
            default:
                break;
        }
    }
}
```

填充完了 Postcard 的信息后，LogisticsCenter 会根据 Postcard 的类型来做不同的操作，如果是 Provider 的话，就会调用 Provider 的初始化方法，并且把 Postcard 设为绿色通道。

如果是 Fragment 的话，那就只把 Postcard 设为绿色通道，如果是其他类型，则不设为绿色通道，这里说的绿色通道，其实就是说 Provider 和 Fragment 是跳过拦截器链的。

```
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    ......
    try {
        LogisticsCenter.completion(postcard);
    } catch (NoRouteFoundException ex) {
    
        // 路由失败，降级处理
        logger.warning(Consts.TAG, ex.getMessage());

        if (debuggable()) {
            // Show friendly tips for user.
            runInMainThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(mContext, "There's no route matched!\n" +
                            " Path = [" + postcard.getPath() + "]\n" +
                            " Group = [" + postcard.getGroup() + "]", Toast.LENGTH_LONG).show();
                }
            });
        }

        if (null != callback) {
            callback.onLost(postcard);
        } else {
            // No callback for this invoke, then we use the global degrade service.
            DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
            if (null != degradeService) {
                degradeService.onLost(context, postcard);
            }
        }

        return null;
    }

    ......    
}    
```

在填充PostCard路由信息的过程中如果遇到异常，则进行降级处理。

![](<../../../.gitbook/assets/image (329).png>)

## 3. 参考

* [开源最佳实践：Android平台页面路由框架ARouter](https://developer.aliyun.com/article/71687)
