# OkHttp-overview

## 1. HTTP请求库

**HttpClient：**

&#x20;       HttpClient是Apache基金会的一个开源网络库，主要在**Android2.2以前的版本**上使用。它的功能十分强大，API数量众多，但正是由于庞大的API数量使得很难在不破坏兼容性的情况下对它进行升级和扩展，Android团队后续在提升和优化HttpClient方面上并不积极，所以在很早就停止维护。

> Apache HttpClient早就不推荐httpclient，Android 5.0之后废弃，并在Android6.0上删除了HttpClient。

**HttpURLConnection：**

&#x20;       HttpURLConnection是一种多用途，轻量极的HTTP客户端， 提供的API比较简单，比较容易地去使用和扩展。在Android 2.2版本之前，HttpURLConnection一直存在着一些bug。比如说对一个可读的InputStream调用close()方法时，就有可能会导致连接池失效了。在**Android 2.3版本**迭代后，对于新的应用程序更加偏向于使用HttpURLConnection。它的API简单，体积较小，因而非常适用于Android项目。压缩和缓存机制可以有效地减少网络访问的流量，在提升速度和省电方面也起到了较大的作用。

> 从Android4.4开始HttpURLConnection的底层实现采用的是okHttp。

**Volley**:

&#x20;       Volley自己的定位是轻量级网络交互，适合大量的，小数据传输。自带缓存，支持自定义请求。不适合大文件上传和下载。Volley在Android 2.3及以上版本，使用的是HttpURLConnection，而在Android 2.2及以下版本，使用的是HttpClient。Volley在功能拓展性上始终无法与OkHttp相比。最后Volley停止了更新，而OkHttp得到了Android官方的认可。

**OkHttp：**

&#x20;       OkHttp是当下Android中主流的网络请求框架，由Square公司开源，支持同步/异步，而且实现了spdy、http2、websocket协议，api很简洁易用，和volley一样实现了http协议的缓存，是目前Android平台上使用最为广泛的开源网络库。在**Android4.4版本**的源码中HttpURLConnection已经替换成OkHttp实现了。

&#x20;       OkHttp的特点：

* 直接基于Socket，支持HTTP/2，websocket，并允许对同一主机的所有请求共享一个套接字。
* 拥有自动维护的socket连接池，减少握手次数/请求延时。
* 拥有队列线程池，轻松实现并发请。
* 默认实现Gzip压缩数据。
* 实现了基于Headers的缓存策略，避免多余的重复请求。
* 请求失败自动重试主机的其他ip，及自动重定向。

## 2. OkHttp基本用法

OkHttp的用法比较简单：

* 首先获取OkHttpClient对象（在构造OkHttpClient过程中可以直接使用默认的Builder进行构造，也可以自定义构造）；
* 通过Builder构造Request对象，包括构造url、请求的方法（get、post）以及body（参数）等；
* 通过调用OkHttpClient中的newCall方法同步或者异步（execut/enqueue）去请求网络；
* 等待处理返回结果(response)；

**GET：**

```
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

**POST：**

```
public static final MediaType JSON
    = MediaType.get("application/json; charset=utf-8");

OkHttpClient client = new OkHttpClient();

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(json, JSON);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

## 3. OkHttp核心类

![OkHttp框架](<../../../.gitbook/assets/image (251).png>)

![](<../../../.gitbook/assets/image (64).png>)

### 3.1 OkHttpClient

&#x20;       OkHttpClient的**核心类**，管理所有内部逻辑对象，Call的工厂。由于OkHttpClinet中涉及到的构造参数/成员非常多，使用Builder建造者模式构造OkHttpClient对象。

```
open class OkHttp internal constructor(Builder builder) {

    constructor() : this(Builder())

    class Builder() constructor(){
        fun build() : OkHttpClient = OkhttpClient(this)
    }
}
```

### 3.1.2 Request/Response

&#x20;       HTTP请求与响应

### 3.1.3 Call(RealCall)

&#x20;       将被执行的HTTP请求，通过okhttpClient.newCall(request) 方法获取一个 Call 对象（实现为 RealCall），然后在通过 Call 的 execute/enqueue 将发起一个同步/异步请求。

&#x20;       **每一个 Request 最终将会被封装成一个 RealCall 对象，RealCall 与 Request 是一一对应的关系**，Call 用来描述一个可被执行、中断的请求，我们每发起一个请求时就会创建一个 RealCall 对象，最终调用 RealCall#getResponseWithInterceptorChain() 发起请求，该方法将返回一个响应结果 Response。

&#x20;       **每个Call只能执行一次，否则会抛出“Already Executed“ IllegalStateException异常。**

### 3.1.4 Dispatcher

&#x20;       Dispatcher 用于管理其对应 OkHttpClient的所有请求，使用异步请求时会将请求委托给 Dispatcer 对象来处理，**Dispatcher 对象随 OkHttpClient 创建而创建**。

&#x20;       实际上，**Dispatcher 不仅用于管理异步请求，也负责管理同步请求**，当我们发起一个请求时，无论是异步还是同步都会被 Dispatcher 记录下来。我们可以通过 OkHtpClient#dispatcher() 获取 Dispatcher 对象对请求进行统一的控制，例如结束所有请求、获取线程池等等。\
Dispatcher 中包含三个队列：

* readyAsyncCalls：一个新的异步请求首先会被加入该队列中
* runningAsyncCalls：当前正在运行中的异步请求
* runningSyncCalls：当前正在运行的同步请求

&#x20;       Dispatcher 中包含一个默认的线程池用于执行所有的异步请求，也可以通过构造器指定一个线程池，所有的异步请求都将会通过这个线程池来执行。异步请求与同步请求一样，**最终也会调用 RealCall#getResponseWithInterceptorChain() 发起请求**，只不过一个是直接调用，一个是在线程池中调用。

## 4. 请求流程

![](<../../../.gitbook/assets/image (154).png>)

&#x20;       Okhttp的整体执行流程非常清晰，通过OkhttpClient创建请求RealCall后，通过Dispatcher调度执行，无论同步或异步，最终都调用到getResponseWithInterceptorChain()中，通过各个拦截器实现请求细节。

```
@Throws(IOException::class)
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  interceptors += client.interceptors
  interceptors += RetryAndFollowUpInterceptor(client)
  interceptors += BridgeInterceptor(client.cookieJar)
  interceptors += CacheInterceptor(client.cache)
  interceptors += ConnectInterceptor
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  interceptors += CallServerInterceptor(forWebSocket)

  val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
  )

  var calledNoMoreExchanges = false
  try {
    val response = chain.proceed(originalRequest)
    if (isCanceled()) {
      response.closeQuietly()
      throw IOException("Canceled")
    }
    return response
  } catch (e: IOException) {
    calledNoMoreExchanges = true
    throw noMoreExchanges(e) as Throwable
  } finally {
    if (!calledNoMoreExchanges) {
      noMoreExchanges(null)
    }
  }
}
```

* 重试拦截器在负责处理请求失败重试，及根据响应码自动重定向。
* 桥接拦截器负责将HTTP协议必备的请求头加入其中(如：Host)并添加一些默认的行为(如：GZIP压缩)；在获得了结果后，调用保存cookie接口并解析GZIP数据。
* 缓存拦截器顾名思义，交出之前读取并判断是否使用缓存；获得结果后判断是否需要进行缓存。
* 连接拦截器负责找到或者新建一个连接，并获得对应的socket流；在获得结果后不进行额外的处理。
* 请求服务器拦截器进行真正的与服务器的通信，向服务器发送数据，解析读取的响应数据。

 

![](<../../../.gitbook/assets/image (182).png>)
