# Retrofit源码分析

Retrofit是一个Restful设计风格的 HTTP 网络请求框架的封装。网络请求实际由 `OkHttp` 完成，而Retrofit 仅负责网络请求接口的封装。

![](<../../../.gitbook/assets/image (240).png>)

### 1. 基本使用 <a href="#ji-ben-shi-yong" id="ji-ben-shi-yong"></a>

一般情况，Retrofit 的使用流程按照以下三步：

1、定义请求接口，将 HTTP API 定义成接口形式

```
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

2、构建 Retrofit 实例，并生成请求接口服务。

```
// Retrofit 构建过程
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

3、执行网络请求，同步或异步。

```
Call<List<Repo>> repos = service.listRepos("xxx");

call.execute() 或者 call.enqueue()
```

### 2. Retrofit 创建过程 <a href="#retrofit-chuang-jian-guo-cheng" id="retrofit-chuang-jian-guo-cheng"></a>

```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();
```

与OkHttp一样，Retrofit的创建也是使用Builder模式。Retrofit中关键的数据成员如下所示：

```
public final class Retrofit {
  // 根据接口中的方法名，存储方法对应的ServiceMethod对象。
  private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

  // 网络请求工厂，创建okhttp3.Call，默认工厂是OkHttp
  final okhttp3.Call.Factory callFactory;

  // 请求的url地址等信息，包括scheme、username、password、
  // host、port、path、query、fragment等信息
  final HttpUrl baseUrl;

  // 数据转换器的工厂的集合，存储数据转换器工厂，来生成数据转换器
  final List<Converter.Factory> converterFactories;

  // 网络请求适配器工厂的集合
  final List<CallAdapter.Factory> callAdapterFactories;

  // 回调方法执行器，在 Android 上默认是封装了 handler 的 MainThreadExecutor,
  // 默认作用是：切换线程（子线程 -> 主线程）
  final @Nullable Executor callbackExecutor;

  // 是否提前创建 ServiceMethod 缓存的开关，
  // 如果为true，则在调retrofit.create的时候就会生成 ServiceMethod 并添加到缓存里面，
  // 如果为false，在请求的时候才会生成 ServiceMethod 并添加到缓存里面。
  final boolean validateEagerly;
}
```

Retrofit 的构造是由它的内部类 Builder 来生成的，在调用 Builder 的 build() 方法的时候，会创建 Retrofit 对象，来看下 Builder 类：

```
public static final class Builder {
  // 当前的平台
  private final Platform platform;
  private @Nullable okhttp3.Call.Factory callFactory;
  private HttpUrl baseUrl;
  private final List<Converter.Factory> converterFactories = new ArrayList<>();
  private final List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
  private @Nullable Executor callbackExecutor;
  private boolean validateEagerly;
	// 上面的一些属性和 Retrofit 中一致,用来给 Retrofit 赋值用
  
  Builder(Platform platform) {
    this.platform = platform;
  	//首先添加内置转换器工厂。这可以防止被覆盖，而且确保使用使用所有类型的转换器时的正确行为
    converterFactories.add(new BuiltInConverters());
  }

  public Builder() {
    this(Platform.get());
  }

  Builder(Retrofit retrofit) {
    platform = Platform.get();
    callFactory = retrofit.callFactory;
    baseUrl = retrofit.baseUrl;
    converterFactories.addAll(retrofit.converterFactories);
    adapterFactories.addAll(retrofit.adapterFactories);
    // Remove the default, platform-aware call adapter added by build().
    adapterFactories.remove(adapterFactories.size() - 1);
    callbackExecutor = retrofit.callbackExecutor;
    validateEagerly = retrofit.validateEagerly;
  }
}
```

在生成 Builder 对象的时候，会调用无参的构造方法，然后获取当前平台，然后再调用有参的构造方法。看下 Platform.get()：

```
class Platform {
  // 类加载的时候就创建好了 Platform
  private static final Platform PLATFORM = findPlatform();
	
  // 获取平台信息
  static Platform get() {
    return PLATFORM;
  }

  // 通过反射获取当前平台，我们是Android，所以返回 Android 平台
  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
}
```

在这里返回的是 Android 平台：

```
static class Android extends Platform {
  // 获取 Android 平台默认的callback 回调处理器，这里返回的是MainThreadExecutor
  // MainThreadExecutor 是 Android 的一个静态内部类，里面初始化了主线程的Handler
  // 也就是最后收到服务器响应以后切换到主线程处理消息。
  @Override public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  // 获取默认的请求适配器工厂，这里使用的 ExecutorCallAdapterFactory。
  // 用于在执行异步请求时候在
  // ExecutorCallAdapterFactory 中生成的Executor 中执行。
  // Retrofit 提供了四种请求适配器，分别为：
  // ExecutorCallAdapterFactory/(Rxjava1/2)/Guava/java8
  // 提供的请求适配器：https://github.com/square/retrofit/wiki/Call-Adapters
  @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor == null) throw new AssertionError();
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }

  static class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```

然后调用了有参的构造方法完成了Builder 对象的创建并添加了内置的转换器工厂。

上面是创建 Builder 对象的过程，在获取完对象并且设置完参数以后，最后调用的 .build() 方法：

```
public Retrofit build() {
  // 这里指定了，我们不能设置为 null 的 baseUrl
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }
	// 这里设置了网络请求工厂，如果没有设置，就使用OkHttpClient作为网络请求工厂
  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }
	// 设置回调方法执行器，如果没有指定，就使用上面Android 平台指定的执行器（也就是回调到主线程）
  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }
	// 获取自定义设置的请求适配器工厂集合（这里是通过addCallAdapterFactory添加的），
	// 添加到 adapterFactories ，然后再把上面platform 
  // 设置的平台默认网络请求适配器工厂也加入adapterFactories
  List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
  adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
	// 获取自定义设置的数据转换器工厂集合（这里是通过addConverterFactory 添加的），
  // 添加到converterFactories，注意此时在创建Builder的时候已经把内置的转换器工厂添加了进去
  List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);
	// 调用Retrofit 的构造方法创建 Retrofit。需要注意的是 validateEagerly 默认是false的，
  // 也就是不提前缓存ServiceMethod。
  return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
      callbackExecutor, validateEagerly);
}
```

### 3. 接口代理创建

```
GitHubService service = retrofit.create(GitHubService.class);
```

通过retrofit.create创建对应的接口代理对象，具体实现如下：

```
public <T> T create(final Class<T> service) {
  // 接口有效性校验
  Utils.validateServiceInterface(service);
  // 这个就是前面说的控制是否会提前获取ServiceMethod 并且缓存的判断，默认是false的
  // 如果设置为 true，则执行 eagerlyValidateMethods
  if (validateEagerly) {
    //提前创建好 MovieService 接口中的所有方法对应的 ServiceMethod 信息
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}

	// 提前创建好 MovieService 接口中的所有方法对应的 ServiceMethod 信息
  private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
    // 反射拿到接口中的所有方法
    for (Method method : service.getDeclaredMethods()) {
      // isDefaultMethod 返回的一直是false ,也就是会执行 loadServiceMethod
      if (!platform.isDefaultMethod(method)) {
        // 循环创建
        loadServiceMethod(method);
      }
    }
  }
```

这里会调用 loadServiceMethod(method) ，loadServiceMethod 方法生成并缓存 ServiceMethod

```
  // 获取 MovieService 接口中某个方法对应的 ServiceMethod 信息
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    // 从缓存serviceMethodCache中取方法对应的ServiceMethod 。
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    // 如果取到 ServiceMethod ，直接返回，继续下一个
    if (result != null) return result;
		// 这里没取到，加上同步锁，
    synchronized (serviceMethodCache) {
      // 继续获取一次，没取到就去创建一个新的 ServiceMethod，
      // 并且添加到缓存 serviceMethodCache中
      result = serviceMethodCache.get(method);
      if (result == null) {
        // 根据method生成对应的ServiceMethod
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    // 返回 ServiceMethod
    return result;
  }	
```

继续看ServiceMethod.parseAnnotations的过程

```
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  　// 解析方法的注解、参数注解、请求头等等
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    // 根据请求方法、信息创建HttpServiceMethod
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```

继续查看 HttpServiceMethod.parseAnnotations() 方法。

```
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {

    //1.根据网络请求接口方法的返回值和注解类型，
    // 从Retrofit对象中获取对应的网络请求适配器
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
        
    // 得到响应类型
    Type responseType = callAdapter.responseType();

    ...

    //2.根据网络请求接口方法的返回值和注解类型从Retrofit对象中获取对应的数据转换器 
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;

    //3.创建CallAdapted
    return new CallAdapted<>(requestFactory, callFactory, 
                             responseConverter, callAdapter);
}
```

createCallAdapter即通过retrofit.callAdapter确定使用的adapter, 这里DefaultCallAdapterFactory

```
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
    Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
  try {
    //noinspection unchecked
    return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create call adapter for %s", returnType);
  }
}
```

HttpServiceMethod.parseAnnotations() 中根据获取的requestFactory, callFactory, responseConverter, callAdapter创建出CallAdapted，CallAdapted是HttpServiceMethod中的内部类，继承于HttpServiceMethod。

```
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
  private final CallAdapter<ResponseT, ReturnT> callAdapter;

  CallAdapted(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, ReturnT> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
  }

  @Override
  protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
    return callAdapter.adapt(call);
  }
}
```

### 4. 请求执行过程

```
Call<List<Repo>> repos = service.listRepos("xxx");

call.execute() 或者 call.enqueue()
```

当调用接口方法时，会直接执行代理对象的InvocationHandler代理方法，即

```
loadServiceMethod(method).invoke(args)
```

即进入HttpServiceMethod.invoke()执行

```
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```

invoke返回的是ReturnT类型，即我们这里的Call\<List\<Repo>>,  以OkHttpCall为参数进入到CallAdapted.adapt

```
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
  private final CallAdapter<ResponseT, ReturnT> callAdapter;

  CallAdapted(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, ReturnT> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
  }

  @Override
  protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
    return callAdapter.adapt(call);
  }
}
```

这里的callAdapter即为前面默认的DefaultCallAdapterFactory.get()

```
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  private final @Nullable Executor callbackExecutor;

  DefaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    if (!(returnType instanceof ParameterizedType)) {
      throw new IllegalArgumentException(
          "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
    }
    final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }

  ....
}

```

即返回ExecutorCallbackCall。

![](<../../../.gitbook/assets/image (364).png>)



![](<../../../.gitbook/assets/image (34).png>)
