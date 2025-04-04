# OkHttp-interceptor1

### 1. RetryAndFollowUpInterceptor      &#x20;

&#x20;RetryAndFollowUpInterceptor是OkHttp中执行的第一个Interceptor，主要负责完成**失败重试及重定向**工&#x4F5C;**。**

&#x20;       RetryAndFollowUpInterceptor.intercept()中的主要处理过程是个while(true)循环，在while(true)循环中判断失败及重定向条件，直至正常返回或者异常退出。主要有以下5步：

1. 进入循环并且初始化ExchangeFinder
2. 判断请求是否已取消，请求取消时立即退出
3. 调用下一个Interceptor执行请求服务器流程。
4. 请求过程中出现RouteException或IOException时判断是否需要重试，需要重试则返回到1，不需要则直接抛出异常退出。
5. 根据3获取的response判断是否需要进行重定向，不需要重定向，直接返回response，结束。
6. 需要重定向则根据重新向信息创建新的请求，继续1。

![](<../../../.gitbook/assets/image (151).png>)

```
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {

  val realChain = chain as RealInterceptorChain
  var request = chain.request
  val call = realChain.call

  // 重定向计数
  var followUpCount = 0
  var priorResponse: Response? = null
  var newExchangeFinder = true
  var recoveredFailures = listOf<IOException>()

  // 在循环中处理失败重试及重定向，当满足重试及重定向条件时重新发起请求。
  // 循环退出条件：
  // 1，call请求被取消
  // 2，正常拿到服务器response后返回
  // 2，出现的异常不可恢复
  // 3，重定向请求超过定义的次数
  //
  while (true) {
    // 在call中初始化ExchangeFinder，用于后续查找可用连接
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)

    var response: Response
    var closeActiveExchange = true
    try {
      // 请求已取消，直接抛出IOException退出
      if (call.isCanceled()) {
        throw IOException("Canceled")
      }

      try {
        // 传递给后面的Chain处理，请求成功将从proceed返回服务器的response。
        response = realChain.proceed(request)
        newExchangeFinder = true
      } catch (e: RouteException) {
        // 请求服务器过程中出现RouteException，通过recover()判断是否是重试可恢复的异常，
        // 此时请求还没有被发送。
        if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
          // 不可恢复，抛出异常退出
          throw e.firstConnectException.withSuppressed(recoveredFailures)
        } else {
          // 记录异常信息
          recoveredFailures += e.firstConnectException
        }
        // 复用最初的ExchangeFinder，回到while(true)中重试
        newExchangeFinder = false
        continue
      }
      catch (e: IOException) {
        // 请求服务器过程中出现IOException，通过recover()判断是否是重试可恢复的异常，
        // 通过是否是ConnectionShutdownException判断请求是否已发送。
        if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
          // 不可恢复，抛出异常退出
          throw e.withSuppressed(recoveredFailures)
        } else {
          // 记录异常信息
          recoveredFailures += e
        }
        // 复用最初的ExchangeFinder，回到while(true)中重试
        newExchangeFinder = false
        continue
      }

      // 重定向时返回的response，这种response没有body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }

      val exchange = call.interceptorScopedExchange
      // 判断是否需要重定向等处理，无需处理则返回null
      val followUp = followUpRequest(response, exchange)

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          call.timeoutEarlyExit()
        }
        closeActiveExchange = false
        // 无需重定向请求，直接返回response
        return response
      }

      val followUpBody = followUp.body
      if (followUpBody != null && followUpBody.isOneShot()) {
        closeActiveExchange = false
        return response
      }

      response.body?.closeQuietly()

      // 超过定义的重定向次数，抛出异常退出
      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }

      // 将重定向请求赋值给request再次请求
      request = followUp
      priorResponse = response
    } finally {
      call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
  }
}
```

&#x20;       判断是否可恢复重试的recover()实现如下：

![](<../../../.gitbook/assets/image (457).png>)

```
// requestSendStarted HTTP/2的时候才有可能是true，HTTP/1是false
private fun recover(
  e: IOException,
  call: RealCall,
  userRequest: Request,
  requestSendStarted: Boolean
): Boolean {
  // OkHttpClient中的失败重试开关关闭（默认打开），无需重试
  if (!client.retryOnConnectionFailure) return false

  // 请求已经发送的情况下，requestIsOneShot不能重复发送请求
  if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

  // 异常不可恢复，无需重试
  if (!isRecoverable(e, requestSendStarted)) return false

  // 没有更多的路由可以重试，无需重试
  if (!call.retryAfterFailure()) return false

  // For failure recovery, use the same route selector with a new connection.
  return true
}
```

         在isRecoverable()中判断不能通过路由跟重试解决的错误：

* ProtocolException异常不重试
* SocketTimeoutException且请求并未发送成功的情况下可重试
* 因为证书异常，SSL握手失败不需要重试，其他情况可重试

```
private fun isRecoverable(e: IOException, requestSendStarted: Boolean): Boolean {
  // If there was a protocol problem, don't recover.
  if (e is ProtocolException) {
    return false
  }

  // If there was an interruption don't recover, but if there was a timeout connecting to a route
  // we should try the next route (if there is one).
  if (e is InterruptedIOException) {
    return e is SocketTimeoutException && !requestSendStarted
  }

  // Look for known client-side or negotiation errors that are unlikely to be fixed by trying
  // again with a different route.
  if (e is SSLHandshakeException) {
    // If the problem was a CertificateException from the X509TrustManager,
    // do not retry.
    if (e.cause is CertificateException) {
      return false
    }
  }
  if (e is SSLPeerUnverifiedException) {
    // e.g. a certificate pinning error.
    return false
  }
  // An example of one we might want to retry with a different route is a problem connecting to a
  // proxy and would manifest as a standard IOException. Unless it is one we know we should not
  // retry, we return true and try a new route.
  return true
}
```

         在followUpRequest方法中，将会根据响应userResponse，获取到响应码，并根据响应码来判断是否需要重定向。需要重定向则返回重组的request，不需要则直接返回null。

| 响应码                      | 说明                                 | 重定向条件                                                                                                                                      |
| ------------------------ | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 407                      | 代理需要授权，如付费代理，需要验证身份                | 通过proxyAuthenticator获 得到了Request。 例: 添加 Proxy-Authorization 请求头                                                                            |
| 401                      | 服务器需要授权，如某些接口需要登陆才能使用 (不安全，基本上没用了) | <p>通过authenticator获得到了Request。<br>例: 添加 Authorization 请求头</p>                                                                              |
| 300、301、302、303、 307、308 | 重定向响应                              | <p>307与308必须为GET/HEAD请求再继续判断<br>1、用户允许自动重定向(默认允许)<br>2、能够获得 <strong>Location</strong> 响应头，并且值为有效url 3、如果重定向需要由HTTP到https间切换，需要允许(默认允许)</p> |
| 408                      | 请求超时。服务器觉得你太慢了                     | <p>1、用户允许自动重试(默认允许)<br>2、本次请求的结果不是 响应408的重试结果<br>3、服务器未响应Retry-After(稍后重试),或者响应Retry-After: 0。</p>                                         |
| 503                      | 服务不可用                              | <p>1、本次请求的结果不是 响应503的重试结果<br>2、服务器明确响应 Retry-After: 0，立即重试</p>                                                                             |

```
@Throws(IOException::class)
private fun followUpRequest(userResponse: Response, exchange: Exchange?): Request? {
  val route = exchange?.connection?.route()
  val responseCode = userResponse.code

  val method = userResponse.request.method
  when (responseCode) {
    // 407 调用OkHttpClient.proxyAuthenticator认证
    HTTP_PROXY_AUTH -> {
      val selectedProxy = route!!.proxy
      if (selectedProxy.type() != Proxy.Type.HTTP) {
        throw ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy")
      }
      return client.proxyAuthenticator.authenticate(route, userResponse)
    }
    // 401调用OkHttpClient.authenticator认证
    HTTP_UNAUTHORIZED -> return client.authenticator.authenticate(route, userResponse)
    // 308 307 300 301 302 303 重定向
    HTTP_PERM_REDIRECT, HTTP_TEMP_REDIRECT, HTTP_MULT_CHOICE, HTTP_MOVED_PERM, HTTP_MOVED_TEMP, HTTP_SEE_OTHER -> {
      return buildRedirectRequest(userResponse, method)
    }
    // 408
    HTTP_CLIENT_TIMEOUT -> {
      // 408's are rare in practice, but some servers like HAProxy use this response code. The
      // spec says that we may repeat the request without modifications. Modern browsers also
      // repeat the request (even non-idempotent ones.)
      if (!client.retryOnConnectionFailure) {
        // The application layer has directed us not to retry the request.
        return null
      }

      val requestBody = userResponse.request.body
      if (requestBody != null && requestBody.isOneShot()) {
        return null
      }
      val priorResponse = userResponse.priorResponse
      if (priorResponse != null && priorResponse.code == HTTP_CLIENT_TIMEOUT) {
        // We attempted to retry and got another timeout. Give up.
        return null
      }

      if (retryAfter(userResponse, 0) > 0) {
        return null
      }

      return userResponse.request
    }
    // 503
    HTTP_UNAVAILABLE -> {
      val priorResponse = userResponse.priorResponse
      if (priorResponse != null && priorResponse.code == HTTP_UNAVAILABLE) {
        // We attempted to retry and got another timeout. Give up.
        return null
      }

      if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {
        // specifically received an instruction to retry without delay
        return userResponse.request
      }

      return null
    }

    // 421
    HTTP_MISDIRECTED_REQUEST -> {
      // OkHttp can coalesce HTTP/2 connections even if the domain names are different. See
      // RealConnection.isEligible(). If we attempted this and the server returned HTTP 421, then
      // we can retry on a different connection.
      val requestBody = userResponse.request.body
      if (requestBody != null && requestBody.isOneShot()) {
        return null
      }

      if (exchange == null || !exchange.isCoalescedConnection) {
        return null
      }

      exchange.connection.noCoalescedConnections()
      return userResponse.request
    }

    else -> return null
  }
}
```

&#x20;       在buildRedirectRequest中重新build定向的request。

```
private fun buildRedirectRequest(userResponse: Response, method: String): Request? {
  // 根据OkHttpClient followRedirects配置判断是否允许重定向，默认为true允许
  if (!client.followRedirects) return null

  // 从Response header的"Location"字段拿到重定向的url
  val location = userResponse.header("Location") ?: return null
  // Don't follow redirects to unsupported protocols.
  val url = userResponse.request.url.resolve(location) ?: return null

  // 根据OkHttpClient followSslRedirects配置，判断是否可以在http与https之间重定向，默认为true允许
  val sameScheme = url.scheme == userResponse.request.url.scheme
  if (!sameScheme && !client.followSslRedirects) return null

  // Most redirects don't include a request body.
  val requestBuilder = userResponse.request.newBuilder()
  if (HttpMethod.permitsRequestBody(method)) {
    val responseCode = userResponse.code
    val maintainBody = HttpMethod.redirectsWithBody(method) ||
        responseCode == HTTP_PERM_REDIRECT ||
        responseCode == HTTP_TEMP_REDIRECT
    if (HttpMethod.redirectsToGet(method) && responseCode != HTTP_PERM_REDIRECT && responseCode != HTTP_TEMP_REDIRECT) {
      requestBuilder.method("GET", null)
    } else {
      val requestBody = if (maintainBody) userResponse.request.body else null
      requestBuilder.method(method, requestBody)
    }
    if (!maintainBody) {
      requestBuilder.removeHeader("Transfer-Encoding")
      requestBuilder.removeHeader("Content-Length")
      requestBuilder.removeHeader("Content-Type")
    }
  }

  // When redirecting across hosts, drop all authentication headers. This
  // is potentially annoying to the application layer since they have no
  // way to retain them.
  if (!userResponse.request.url.canReuseConnectionFor(url)) {
    requestBuilder.removeHeader("Authorization")
  }

  // 根据重定向url创建新的request
  return requestBuilder.url(url).build()
}
```

### 2. BridgeInterceptor  &#x20;

&#x20;       BridgeInterceptor是请求网络过程中执行的第二个Interceptor，主要负责对请求头添加一些字段，补全请求头，并对返回的response进行gzip解压等操作。

| 请求头                              | 说明                                        |
| -------------------------------- | ----------------------------------------- |
| Content-Type                     | 请求体类型，如：application/x-www-form-urlencoded |
| Content-Length/Transfer-Encoding | 请求体解析方式                                   |
| Host                             | 请求的主机                                     |
| Connection: Keep-Alive           | 默认保持长连接                                   |
| Accept-Encoding: gzip            | 接收响应体使用gzip压缩                             |
| Cookie                           | 保存通信状态，默认没有实现                             |
| User-Agent                       | 用户信息，如：操作系统、浏览器等                          |

![](<../../../.gitbook/assets/image (170).png>)

```
class BridgeInterceptor(private val cookieJar: CookieJar) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val userRequest = chain.request()
    val requestBuilder = userRequest.newBuilder()

    val body = userRequest.body
    if (body != null) {
      val contentType = body.contentType()
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString())
      }

      // contentLength不能指定时采用Transfer-Encoding分块编码
      val contentLength = body.contentLength()
      if (contentLength != -1L) {
        requestBuilder.header("Content-Length", contentLength.toString())
        requestBuilder.removeHeader("Transfer-Encoding")
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked")
        requestBuilder.removeHeader("Content-Length")
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", userRequest.url.toHostHeader())
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive")
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    var transparentGzip = false
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true
      requestBuilder.header("Accept-Encoding", "gzip")
    }

    val cookies = cookieJar.loadForRequest(userRequest.url)
    if (cookies.isNotEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies))
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", userAgent)
    }

    val networkResponse = chain.proceed(requestBuilder.build())

    cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

    val responseBuilder = networkResponse.newBuilder()
        .request(userRequest)

    // 收到response后，gzip解压以后返回response
    if (transparentGzip &&
        "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
        networkResponse.promisesBody()) {
      val responseBody = networkResponse.body
      if (responseBody != null) {
        val gzipSource = GzipSource(responseBody.source())
        val strippedHeaders = networkResponse.headers.newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build()
        responseBuilder.headers(strippedHeaders)
        val contentType = networkResponse.header("Content-Type")
        responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
      }
    }

    return responseBuilder.build()
  }

  /** Returns a 'Cookie' HTTP request header with all cookies, like `a=b; c=d`. */
  private fun cookieHeader(cookies: List<Cookie>): String = buildString {
    cookies.forEachIndexed { index, cookie ->
      if (index > 0) append("; ")
      append(cookie.name).append('=').append(cookie.value)
    }
  }
}
```

###  3. CacheInterceptor

&#x20;       CacheInterceptor负责缓存处理，包括在发出请求前判断是否命中缓存，命中则直接使用缓存，无需向响应。 并在获取到服务器response后判断是否需要缓存。(只会存在Get请求的缓存)

&#x20;       具体的步骤为:

* 从缓存中获得对应请求的响应缓存
* 创建`CacheStrategy` ,通过它判断是否能够使用缓存，在`CacheStrategy` 中存在两个成员:`networkRequest`与`cacheResponse`。组合如下:

| networkRequest | cacheResponse | 说明                               |
| -------------- | ------------- | -------------------------------- |
| Null           | Not Null      | networkRequest为null，并且有缓存，直接使用缓存 |
| Not Null       | Null          | 没有可用缓存，直接向服务器发起请求                |
| Null           | Null          | 都为null，504错误结束请求                 |
| Not Null       | Not Null      | 发起请求，若得到响应为304(无修改)，则更新缓存响应并返回   |

* 交给下一个责任链条，请求服务器并获取到响应
* 获取到的响应返回304则使用缓存，否则使用网络响应并缓存本次响应（只缓存Get请求的响应）

```
class CacheInterceptor(internal val cache: Cache?) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val call = chain.call()

    // 通过url查找缓存（只有GET请求才有缓存）
    val cacheCandidate = cache?.get(chain.request())

    val now = System.currentTimeMillis()

    // 根据各种条件(请求头)获取缓存策略
    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    // 后续根据networkRequest与cacheResponse是否为空判断是使用缓存还是请求网络
    val networkRequest = strategy.networkRequest
    val cacheResponse = strategy.cacheResponse

    cache?.trackResponse(strategy)
    val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

    if (cacheCandidate != null && cacheResponse == null) {
      // The cache candidate wasn't applicable. Close it.
      cacheCandidate.body?.closeQuietly()
    }

    // networkRequest，cacheResponse都为空，不能进行网络请求也没有缓存使用，返回504
    if (networkRequest == null && cacheResponse == null) {
      return Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(HTTP_GATEWAY_TIMEOUT)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build().also {
            listener.satisfactionFailure(call, it)
          }
    }

    // networkRequest为空，直接使用缓存
    if (networkRequest == null) {
      return cacheResponse!!.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build().also {
            listener.cacheHit(call, it)
          }
    }

    if (cacheResponse != null) {
      listener.cacheConditionalHit(call, cacheResponse)
    } else if (cache != null) {
      listener.cacheMiss(call)
    }

    // 传递到下一个链条，发起请求
    var networkResponse: Response? = null
    try {
      networkResponse = chain.proceed(networkRequest)
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        cacheCandidate.body?.closeQuietly()
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      // 由本地缓存，且服务器返回304，服务器对应资源也无修改，那就使用缓存的响应（修改了时间等数据后）作为本次请求的响应
      if (networkResponse?.code == HTTP_NOT_MODIFIED) {
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build()

        networkResponse.body!!.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        cache.update(cacheResponse, response)
        return response.also {
          listener.cacheHit(call, it)
        }
      } else {
        cacheResponse.body?.closeQuietly()
      }
    }

    // 缓存过期或不可用，使用网络的响应
    val response = networkResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build()

    // 缓存获取到的response
    if (cache != null) {
      // 获取到的response有body且允许缓存，缓存获取到的response
      if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        val cacheRequest = cache.put(response)
        return cacheWritingResponse(cacheRequest, response).also {
          if (cacheResponse != null) {
            // This will log a conditional cache miss only.
            listener.cacheMiss(call)
          }
        }
      }

      if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
          cache.remove(networkRequest)
        } catch (_: IOException) {
          // The cache cannot be written.
        }
      }
    }

    return response
  }
  
  ......
}  
```

```
// 根据各种条件(请求头)获取缓存策略
val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
```

         是否使用缓存根据响应与请求头来判断，下面是请求/响应与缓存头相关的字段。

**响应头：**

| 响应头           | 说明                            | 例子                                           |
| ------------- | ----------------------------- | -------------------------------------------- |
| Date          | 消息发送的时间                       | Date: Sat, 18 Nov 2028 06:17:41 GMT          |
| Expires       | 资源过期的时间                       | Expires: Sat, 18 Nov 2028 06:17:41 GMT       |
| Last-Modified | 资源最后修改时间                      | Last-Modified: Fri, 22 Jul 2016 02:57:17 GMT |
| ETag          | 资源在服务器的唯一标识                   | ETag: "16df0-5383097a03d40"                  |
| Age           | 服务器用缓存响应请求，该缓存从产生到现在经过多长时间(秒) | Age: 3825683                                 |
| Cache-Control | -                             | -                                            |

 **请求头：**

| 请求头                 | 说明                               | 例子                                               |
| ------------------- | -------------------------------- | ------------------------------------------------ |
| `If-Modified-Since` | 服务器没有在指定的时间后修改请求对应资源,返回304(无修改)  | If-Modified-Since: Fri, 22 Jul 2016 02:57:17 GMT |
| `If-None-Match`     | 服务器将其与请求对应资源的`Etag`值进行比较，匹配返回304 | If-None-Match: "16df0-5383097a03d40"             |
| `Cache-Control`     | -                                | -                                                |

其中`Cache-Control`可以在请求头存在，也能在响应头存在，对应的value可以设置多种组合：

* `max-age=[秒]` ：资源最大有效时间;
* `public` ：表明该资源可以被任何用户缓存，比如客户端，代理服务器等都可以缓存资源;
* `private`：表明该资源只能被单个用户缓存，默认是private。
* `no-store`：资源不允许被缓存
* `no-cache`：(请求)不使用缓存
* `immutable`：(响应)资源不会改变
* `min-fresh=[秒]`：(请求)缓存最小新鲜度(用户认为这个缓存有效的时长)
* `must-revalidate`：(响应)不允许使用过期缓存
* `max-stale=[秒]`：(请求)缓存过期后多久内仍然有效

> 假设存在max-age=100，min-fresh=20。这代表了用户认为这个缓存的响应，从服务器创建响应 到 能够缓存使用的时间为100-20=80s。但是如果max-stale=100。这代表了缓存有效时间80s过后，仍然允许使用100s，可以看成缓存有效时长为180s。

```
/** Returns a strategy to satisfy [request] using [cacheResponse]. */
fun compute(): CacheStrategy {
  val candidate = computeCandidate()

  // 需要进行网络请求，但是请求头里面又包含"only-if-cached"，返回networkRequest，cacheResponse
  // 都为空的CacheStrategy，后续直接抛出504结束整个请求流程。
  if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
    return CacheStrategy(null, null)
  }

  return candidate
}
```

```
/** Returns a strategy to use assuming the request can use the network. */
private fun computeCandidate(): CacheStrategy {
  // 没有缓存的response，需要向服务器请求
  if (cacheResponse == null) {
    return CacheStrategy(request, null)
  }

  // 是https请求，但是缓存的response没有握手信息，需要向服务器请求
  if (request.isHttps && cacheResponse.handshake == null) {
    return CacheStrategy(request, null)
  }

  // If this response shouldn't have been stored, it should never be used as a response source.
  // This check should be redundant as long as the persistence store is well-behaved and the
  // rules are constant.
  // 通过响应码以及头部缓存控制字段判断响应能不能缓存，不能缓存那就进行网络请求
  if (!isCacheable(cacheResponse, request)) {
    return CacheStrategy(request, null)
  }

  // 如果请求头包含：CacheControl:no-cache / If-Modified-Since / If-None-Match 三者中任意一个
  // 需要进行网络请求
  val requestCaching = request.cacheControl
  if (requestCaching.noCache || hasConditions(request)) {
    return CacheStrategy(request, null)
  }

  val responseCaching = cacheResponse.cacheControl

  // 缓存的响应从创建到现在的时间
  val ageMillis = cacheResponseAge()
  // 响应有效缓存的时长
  var freshMillis = computeFreshnessLifetime()

  // 如果请求中指定了 max-age 表示指定了能拿的缓存有效时长，就需要综合响应有效缓存时长与请求能拿缓存的时长，获得最小的能够使用响应缓存的时长
  if (requestCaching.maxAgeSeconds != -1) {
    freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
  }
  // 请求包含  Cache-Control:min-fresh=[秒]  能够使用还未过指定时间的缓存 （请求认为的缓存有效时间）
  var minFreshMillis: Long = 0
  if (requestCaching.minFreshSeconds != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
  }

  var maxStaleMillis: Long = 0
  if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
  }
  // 不需要与服务器验证有效性 && 响应存在的时间+请求认为的缓存有效时间 小于 缓存有效时长+过期后还可以使用的时间允许使用缓存
  if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    val builder = cacheResponse.newBuilder()
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
    }
    val oneDayMillis = 24 * 60 * 60 * 1000L
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
    }
    return CacheStrategy(null, builder.build())
  }

  // 缓存过期
  val conditionName: String
  val conditionValue: String?
  when {
    etag != null -> {
      conditionName = "If-None-Match"
      conditionValue = etag
    }

    lastModified != null -> {
      conditionName = "If-Modified-Since"
      conditionValue = lastModifiedString
    }

    servedDate != null -> {
      conditionName = "If-Modified-Since"
      conditionValue = servedDateString
    }

    else -> return CacheStrategy(request, null) // No condition! Make a regular request.
  }

  // 设置了If-None-Match/If-Modified-Since，如果服务器是可能返回304(无修改), 使用缓存的响应体
  val conditionalRequestHeaders = request.headers.newBuilder()
  conditionalRequestHeaders.addLenient(conditionName, conditionValue!!)

  val conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build()
  return CacheStrategy(conditionalRequest, cacheResponse)
}
```

```
fun isCacheable(response: Response, request: Request): Boolean {
  // 200, 203, 204, 300, 301, 404, 405, 410, 414, 501, 308 判断是不是存在cache-control:nostore，存在则不缓存
  // 302, 307 存在: Expires:时间、CacheControl:max-age/public/private 才判断是不是存在cache-control:nostore
  // 其他响应码不缓存
  when (response.code) {
    HTTP_OK,
    HTTP_NOT_AUTHORITATIVE,
    HTTP_NO_CONTENT,
    HTTP_MULT_CHOICE,
    HTTP_MOVED_PERM,
    HTTP_NOT_FOUND,
    HTTP_BAD_METHOD,
    HTTP_GONE,
    HTTP_REQ_TOO_LONG,
    HTTP_NOT_IMPLEMENTED,
    StatusLine.HTTP_PERM_REDIRECT -> {
      // These codes can be cached unless headers forbid it.
    }

    HTTP_MOVED_TEMP,
    StatusLine.HTTP_TEMP_REDIRECT -> {
      // These codes can only be cached with the right response headers.
      // http://tools.ietf.org/html/rfc7234#section-3
      // s-maxage is not checked because OkHttp is a private cache that should ignore s-maxage.
      if (response.header("Expires") == null &&
          response.cacheControl.maxAgeSeconds == -1 &&
          !response.cacheControl.isPublic &&
          !response.cacheControl.isPrivate) {
        return false
      }
    }

    else -> {
      // All other codes cannot be cached.
      return false
    }
  }

  // A 'no-store' directive on request or response prevents the response from being cached.
  return !response.cacheControl.noStore && !request.cacheControl.noStore
}
```

**总结：**&#x0;

1. 如果从缓存获取的`Response`是null，那就需要使用网络请求获取响应；
2. 如果是Https请求，但是又丢失了握手信息，那也不能使用缓存，需要进行网络请求；&#x20;
3. 如果判断响应码不能缓存且响应头有`no-store`标识，那就需要进行网络请求；&#x20;
4. 如果请求头有`no-cache`标识或者有`If-Modified-Since/If-None-Match`，那么需要进行网络请求;
5. 如果响应头没有`no-cache`标识，且缓存时间没有超过极限时间，那么可以使用缓存，不需要进行网络请求；&#x20;
6. 如果缓存过期了，判断响应头是否设置`Etag/Last-Modified/Date`，没有那就直接使用网络请求否则需要考虑服务器返回304；
