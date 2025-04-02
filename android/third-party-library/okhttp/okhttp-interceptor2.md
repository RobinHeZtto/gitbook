# OkHttp-interceptor2

### 1. ConnectInterceptor

接上篇继续分析Okhttp的ConnectInterceptor拦截器，connectInterceptor应该是最重要的拦截器，它同时负责了**获取可用连接**、**Dns解析**和**进行Socket连接**（包括tls连接）等工作。

ConnectInterceptor定义如下：

```
>>> okhttp3.internal.connection

object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    return connectedChain.proceed(realChain.request)
  }
}
```

ConnectInterceptor中调用Realcall.initExchange获取Exchange对象，然后执行下一个拦截器。

```
>>> okhttp3.internal.connection.RealCall

  /** Finds a new or pooled connection to carry a forthcoming request and response. */
  internal fun initExchange(chain: RealInterceptorChain): Exchange {
    synchronized(this) {
      check(expectMoreExchanges) { "released" }
      check(!responseBodyOpen)
      check(!requestBodyOpen)
    }

    // exchangeFinder对象是在RetryAndFollowUpInterceptor中创建的
    val exchangeFinder = this.exchangeFinder!!
    // 在连接池中找到或者创建一个连接
    // codec ExchangeCodec接口，它是一个socket链接，用来传输request和response，
    // 有两种实现：Http1ExchangeCodec ，Http2ExchangeCodec 
    val codec = exchangeFinder.find(client, chain)
    // 创建了一个Exchange，它负责单个http请求和响应对，真正的实现是在codec中（实际的IO操作）
    val result = Exchange(this, eventListener, exchangeFinder, codec)
    // 保存Exchange对象到RealCall中
    this.interceptorScopedExchange = result
    this.exchange = result
    synchronized(this) {
      this.requestBodyOpen = true
      this.responseBodyOpen = true
    }

    if (canceled) throw IOException("Canceled")
    return result
  }
  
// 这个函数中创建ExchangeFinder，在RetryAndFollowUpInterceptor中调用  
fun enterNetworkInterceptorExchange(request: Request, newExchangeFinder: Boolean) {
  check(interceptorScopedExchange == null)

  synchronized(this) {
    check(!responseBodyOpen) {
      "cannot make a new request because the previous response is still open: " +
          "please call response.close()"
    }
    check(!requestBodyOpen)
  }

  if (newExchangeFinder) {
    // 创建一个ExchangeFinder，在它里面实现了Dns，代理选择，
    // 路由选择，tls连接，socket连接
    this.exchangeFinder = ExchangeFinder(
        connectionPool,
        createAddress(request.url),
        this,
        eventListener
    )
  }
}
```

initExchange中通过exchangeFinder在连接池中找到或新建一个连接，然后封装成exchange复制给Realcall成员并返回。其中exchangeFinder在RetryAndFollowUpInterceptor中调用enterNetworkInterceptorExchange创建（注意address在此时创建）。

```
fun find(
  client: OkHttpClient,
  chain: RealInterceptorChain
): ExchangeCodec {
  try {
    val resultConnection = findHealthyConnection(
        connectTimeout = chain.connectTimeoutMillis,
        readTimeout = chain.readTimeoutMillis,
        writeTimeout = chain.writeTimeoutMillis,
        pingIntervalMillis = client.pingIntervalMillis,
        connectionRetryEnabled = client.retryOnConnectionFailure,
        doExtensiveHealthChecks = chain.request.method != "GET"
    )
    // resultConnection 是 RealConnection 类型，它负责建立socket链接 ，
    // 建立tls链接，建立Tunnel（通过http代理，访问https 服务器）等操作
    // 面对http1和http2，又有各自不同的传输方式，所以使用newCodec()抽象为ExchangeCodec，
    // http1和http2 实现不同的接口功能（策略者模式）
    return resultConnection.newCodec(client, chain)
  } catch (e: RouteException) {
    trackFailure(e.lastConnectException)
    throw e
  } catch (e: IOException) {
    trackFailure(e)
    throw RouteException(e)
  }
}
```

继续进入到findHealthyConnection中执行

```
  @Throws(IOException::class)
  private fun findHealthyConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    doExtensiveHealthChecks: Boolean
  ): RealConnection {
    //这里是一个死循环，退出循环的两种情况：1、得到一个健康的连接 2、抛出一个异常
    while (true) {
      val candidate = findConnection(
          connectTimeout = connectTimeout,
          readTimeout = readTimeout,
          writeTimeout = writeTimeout,
          pingIntervalMillis = pingIntervalMillis,
          connectionRetryEnabled = connectionRetryEnabled
      )

      // 确认连接是否是正常的
      if (candidate.isHealthy(doExtensiveHealthChecks)) {
        return candidate
      }

      // 如果连接，不可用，就从连接池中删除
      candidate.noNewExchanges()

      // Make sure we have some routes left to try. One example where we may exhaust all the routes
      // would happen if we made a new connection and it immediately is detected as unhealthy.
      if (nextRouteToTry != null) continue

      val routesLeft = routeSelection?.hasNext() ?: true
      if (routesLeft) continue

      val routesSelectionLeft = routeSelector?.hasNext() ?: true
      if (routesSelectionLeft) continue

      throw IOException("exhausted all routes")
    }
  }

```

上面涉及的类比较多，来几张图，从总体的认识一下

![](<../../../.gitbook/assets/image (91).png>)

RouteSelector 是用来根据代理创建Route， 然后封装到Selection，Selection 针对某一个代理的一系列的Route，切换代理后，该对象也会跟着改变。Route 一个Url可能会对应多个IP地址（DNS负载均衡），每个socket 对应一个Route 对象。Address 包含一个Url，指定的Proxy（可为空）， ProxySelector （可能会有许多Proxy）

![](<../../../.gitbook/assets/image (242).png>)

继续看findConnection的执行过程。

```
  @Throws(IOException::class)
  private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
  ): RealConnection {
    //如果当前的请求已经取消，就抛出异常
    if (call.isCanceled()) throw IOException("Canceled")

    //先尝试使用RealCall对象当前的连接
    val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
    if (callConnection != null) {
      var toClose: Socket? = null
      synchronized(callConnection) {
        if (callConnection.noNewExchanges || !sameHostAndPort(callConnection.route().address.url)) {
          toClose = call.releaseConnectionNoEvents()
        }
      }

      // 从当前RealCall对象获取到连接
      if (call.connection != null) {
        check(toClose == null)
        return callConnection
      }

      // The call's connection was released.
      toClose?.closeQuietly()
      eventListener.connectionReleased(call, callConnection)
    }

    // We need a new connection. Give it fresh stats.
    refusedStreamCount = 0
    connectionShutdownCount = 0
    otherFailureCount = 0

    // 上面RealCall对象的连接不可用，尝试从连接池获取连接
    if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }

  	// 在连接池中也没有找到可用的，就去找Route，然后用Route创建一个连接
  	// 寻找Route的步骤：
  	// 1、先看nextRouteToTry 有没有值，（如果有值，就是上一次循环赋值的），拿出来使用
  	// 2、nextRouteToTry 为空，去Selection 中去找（Selection 就是一些列Route的集合）
  	// 3、如果Selection 中也没有，就去RouteSelector中，先找一个Selection，再在其中找Route
  	// 如果是第一次进来，那么会到第三种情况，先创建一个RouteSelector，next() 后Selection，Route 就都有了
    val routes: List<Route>?
    val route: Route
    if (nextRouteToTry != null) {
      // Use a route from a preceding coalesced connection.
      routes = null
      route = nextRouteToTry!!
      nextRouteToTry = null
    } else if (routeSelection != null && routeSelection!!.hasNext()) {
      // Use a route from an existing route selection.
      routes = null
      route = routeSelection!!.next()
    } else {
    　//获取route的过程其实就是DNS获取到域名IP的过程，这是一个阻塞的过程，会等待DNS结果返回
      // Compute a new route selection. This is a blocking operation!
      var localRouteSelector = routeSelector
      if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
      }
      val localRouteSelection = localRouteSelector.next()
      routeSelection = localRouteSelection
      routes = localRouteSelection.routes

      if (call.isCanceled()) throw IOException("Canceled")

      // Now that we have a set of IP addresses, make another attempt at getting a connection from
      // the pool. We have a better chance of matching thanks to connection coalescing.
      if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
        val result = call.connection!!
        eventListener.connectionAcquired(call, result)
        return result
      }

      route = localRouteSelection.next()
    }

    // 代码如果执行到这里，就说明还没有找到可用连接，但是找到了Route
    val newConnection = RealConnection(connectionPool, route)
    call.connectionToCancel = newConnection
    try {
      // 建立socket 连接，建立tls连接，建立tunnel
      newConnection.connect(
          connectTimeout,
          readTimeout,
          writeTimeout,
          pingIntervalMillis,
          connectionRetryEnabled,
          call,
          eventListener
      )
    } finally {
      call.connectionToCancel = null
    }
    call.client.routeDatabase.connected(newConnection.route())

    // 最后一个参数是true，查找是否有多路复用（http2）的可用链接，如果有的话，就使用，并刚刚新创建的链接
    if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
      //把上面找到的route保存起来，如果当前返回的连接不健康，下次就尝试这个route
      val result = call.connection!!
      nextRouteToTry = route
      newConnection.socket().closeQuietly()
      eventListener.connectionAcquired(call, result)
      return result
    }

    synchronized(newConnection) {
      //放入连接池中
      connectionPool.put(newConnection)
      call.acquireConnectionNoEvents(newConnection)
    }

    eventListener.connectionAcquired(call, newConnection)
    return newConnection
  }
```

调用connect创建socket连接

```
>>> okhttp3.internal.connection.RealConnection

fun connect(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    call: Call,
    eventListener: EventListener
  ) {
    check(protocol == null) { "already connected" }

    var routeException: RouteException? = null
    //连接的规格，默认是TLS（https）和 CLEARTEXT（http）
    val connectionSpecs = route.address.connectionSpecs
    val connectionSpecSelector = ConnectionSpecSelector(connectionSpecs)

    if (route.address.sslSocketFactory == null) {
      //如果没有sslSocketFactory，也就是不能创建ssl连接，那么只能明文连接
      if (ConnectionSpec.CLEARTEXT !in connectionSpecs) {
        //CLEARTEXT 不再规格里，就抛出异常
        throw RouteException(UnknownServiceException(
            "CLEARTEXT communication not enabled for client"))
      }
      val host = route.address.url.host
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        //系统不支持该host 明文传输 
        throw RouteException(UnknownServiceException(
            "CLEARTEXT communication to $host not permitted by network security policy"))
      }
    } else {
      if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
        throw RouteException(UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"))
      }
    }

    while (true) {
      try {
        
        if (route.requiresTunnel()) {
          //创建tunnel，用于通过http代理访问https
          //其中包含connectSocket、createTunnel
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break
          }
        } else {
          //如果不需要创建tunnel，就建立socket 连接，见代码十一
          connectSocket(connectTimeout, readTimeout, call, eventListener)
        }
        //建立tls连接，
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
        eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
        break
      } catch (e: IOException) {
          ...省略代码...
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {
      throw RouteException(ProtocolException(
          "Too many tunnel connections attempted: $MAX_TUNNEL_ATTEMPTS"))
    }

    idleAtNs = System.nanoTime()
  }
```

```
>>> okhttp3.internal.connection.RealConnection

  /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  @Throws(IOException::class)
  private fun connectSocket(
    connectTimeout: Int,
    readTimeout: Int,
    call: Call,
    eventListener: EventListener
  ) {
    val proxy = route.proxy
    val address = route.address
    //创建socket
    val rawSocket = when (proxy.type()) {
      Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
      else -> Socket(proxy)
    }
    this.rawSocket = rawSocket

    eventListener.connectStart(call, route.socketAddress, proxy)
    rawSocket.soTimeout = readTimeout
    try {
      // 调用各平台的socket连接，平台是指：OpenJDK、JDK、Android等
      Platform.get().connectSocket(rawSocket, route.socketAddress, connectTimeout)
    } catch (e: ConnectException) {
      throw ConnectException("Failed to connect to ${route.socketAddress}").apply {
        initCause(e)
      }
    }
    try {
      //newCodec ，创建 Http1ExchangeCodec 会把下面的输入 输出流 传递进去
      //得到输入流，以后往这个流中写入数据，write，就会进行发送
      source = rawSocket.source().buffer()
      //输出流，以后可以在这个流中读取数据
      sink = rawSocket.sink().buffer()

    } catch (npe: NullPointerException) {
      if (npe.message == NPE_THROW_WITH_NULL) {
        throw IOException(npe)
      }
    }
  }
```

如果是https连接，执行connectTls()

```
>>> okhttp3.internal.connection.RealConnection

@Throws(IOException::class)
  private fun connectTls(connectionSpecSelector: ConnectionSpecSelector) {
    val address = route.address
    val sslSocketFactory = address.sslSocketFactory
    var success = false
    var sslSocket: SSLSocket? = null
    try {
      // Create the wrapper over the connected socket.
      sslSocket = sslSocketFactory!!.createSocket(
          rawSocket, address.url.host, address.url.port, true /* autoClose */) as SSLSocket

      // Configure the socket's ciphers, TLS versions, and extensions.
      val connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket)
      if (connectionSpec.supportsTlsExtensions) {
        Platform.get().configureTlsExtensions(sslSocket, address.url.host, address.protocols)
      }

      // Force handshake. This can throw!
      sslSocket.startHandshake()
      // block for session establishment
      val sslSocketSession = sslSocket.session
      val unverifiedHandshake = sslSocketSession.handshake()

      // Verify that the socket's certificates are acceptable for the target host.
      // 默认使用 OkHostnameVerifier 来验证证书和Url是否匹配
      if (!address.hostnameVerifier!!.verify(address.url.host, sslSocketSession)) {
        val peerCertificates = unverifiedHandshake.peerCertificates
        if (peerCertificates.isNotEmpty()) {
          val cert = peerCertificates[0] as X509Certificate
          throw SSLPeerUnverifiedException("""
              |Hostname ${address.url.host} not verified:
              |    certificate: ${CertificatePinner.pin(cert)}
              |    DN: ${cert.subjectDN.name}
              |    subjectAltNames: ${OkHostnameVerifier.allSubjectAltNames(cert)}
              """.trimMargin())
        } else {
          throw SSLPeerUnverifiedException(
              "Hostname ${address.url.host} not verified (no certificates)")
        }
      }
      val certificatePinner = address.certificatePinner!!
      //把tls返回的信息保存起来
      handshake = Handshake(unverifiedHandshake.tlsVersion, unverifiedHandshake.cipherSuite,
          unverifiedHandshake.localCertificates) {
        certificatePinner.certificateChainCleaner!!.clean(unverifiedHandshake.peerCertificates,
            address.url.host)
      }

      // Check that the certificate pinner is satisfied by the certificates presented.
      certificatePinner.check(address.url.host) {
        handshake!!.peerCertificates.map { it as X509Certificate }
      }

      // Success! Save the handshake and the ALPN protocol.
      val maybeProtocol = if (connectionSpec.supportsTlsExtensions) {
        Platform.get().getSelectedProtocol(sslSocket)
      } else {
        null
      }
      socket = sslSocket
      //把输入和输出流保存起来
      source = sslSocket.source().buffer()
      sink = sslSocket.sink().buffer()
      protocol = if (maybeProtocol != null) Protocol.get(maybeProtocol) else Protocol.HTTP_1_1
      success = true
    } finally {
      if (sslSocket != null) {
        Platform.get().afterHandshake(sslSocket)
      }
      if (!success) {
        sslSocket?.closeQuietly()
      }
    }
  }
```

最后，创建ExchangeCodec对象。

```
>>> okhttp3.internal.connection.RealConnection

  @Throws(SocketException::class)
  internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
    //socket 连接
    val socket = this.socket!!
    //输入流
    val source = this.source!!
    //输出流
    val sink = this.sink!!
    //http2 独有的连接
    val http2Connection = this.http2Connection

    //根据 不同的协议，返回不同的 ExchangeCodec
    return if (http2Connection != null) {
      Http2ExchangeCodec(client, this, chain, http2Connection)
    } else {
      socket.soTimeout = chain.readTimeoutMillis()
      source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
      sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
      Http1ExchangeCodec(client, this, source, sink)
    }
  }
```

### 2. CallServerInterceptor

`CallServerInterceptor`，利用`HttpCodec`发出请求到服务器并且解析生成`Response`。

```
//与服务器交互
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange!!
    val request = realChain.request
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()

    var invokeStartEvent = true
    var responseBuilder: Response.Builder? = null
    var sendRequestException: IOException? = null
    try {
      // 发送请求行
      exchange.writeRequestHeaders(request)

      if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
        // 如果请求头存在"Expect: 100-continue"，先发送header，收到100后再发送body
        if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
          exchange.flushRequest()
          responseBuilder = exchange.readResponseHeaders(expectContinue = true)
          exchange.responseHeadersStart()
          invokeStartEvent = false
        }
        if (responseBuilder == null) {
          if (requestBody.isDuplex()) {
            // Prepare a duplex body so that the application can send a request body later.
            exchange.flushRequest()
            val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
            requestBody.writeTo(bufferedRequestBody)
          } else {
            // Write the request body if the "Expect: 100-continue" expectation was met.
            val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
            requestBody.writeTo(bufferedRequestBody)
            bufferedRequestBody.close()
          }
        } else {
          exchange.noRequestBody()
          if (!exchange.connection.isMultiplexed) {
            // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
            // from being reused. Otherwise we're still obligated to transmit the request body to
            // leave the connection in a consistent state.
            exchange.noNewExchangesOnConnection()
          }
        }
      } else {
        exchange.noRequestBody()
      }

      if (requestBody == null || !requestBody.isDuplex()) {
        exchange.finishRequest()
      }
    } catch (e: IOException) {
      if (e is ConnectionShutdownException) {
        throw e // No request was sent so there's no response to read.
      }
      if (!exchange.hasFailure) {
        throw e // Don't attempt to read the response; we failed to send the request.
      }
      sendRequestException = e
    }

    try {
      if (responseBuilder == null) {
        // 读取响应
        responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
        if (invokeStartEvent) {
          exchange.responseHeadersStart()
          invokeStartEvent = false
        }
      }
      var response = responseBuilder
          .request(request)
          .handshake(exchange.connection.handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
      var code = response.code
      if (code == 100) {
        // 如果响应是100，这代表了是请求`Expect: 100-continue`成功的响应，
        // 需要马上再次读取一份响应头，这才是真正的请求对应结果响应头。
        responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
        if (invokeStartEvent) {
          exchange.responseHeadersStart()
        }
        response = responseBuilder
            .request(request)
            .handshake(exchange.connection.handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build()
        code = response.code
      }

      exchange.responseHeadersEnd(response)

      response = if (forWebSocket && code == 101) {
        // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
        response.newBuilder()
            .body(EMPTY_RESPONSE)
            .build()
      } else {
        response.newBuilder()
            .body(exchange.openResponseBody(response))
            .build()
      }

      // Connection:close 服务端不允许/不支持长连接，改连接不能再被复用
      if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
          "close".equals(response.header("Connection"), ignoreCase = true)) {
        exchange.noNewExchangesOnConnection()
      }

      // 204 205状态码的返回没有body，如果有Content-Length大于0，将抛出ProtocolException异常
      if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
        throw ProtocolException(
            "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
      }
      return response
    } catch (e: IOException) {
      if (sendRequestException != null) {
        sendRequestException.addSuppressed(e)
        throw sendRequestException
      }
      throw e
    }
  }
}
```

### &#x20;3.**Interceptors和NetworkInterceptors的区别**

前面提到，在OkHttpClient.Builder的构造方法有两个参数，使用者可以通过addInterceptor 和 addNetworkdInterceptor 添加自定义的拦截器。从前面添加拦截器的顺序可以知道 Interceptors 和 networkInterceptors 刚好一个在 RetryAndFollowUpInterceptor 的前面，一个在后面。

结合前面的责任链调用，假如一个请求在 RetryAndFollowUpInterceptor 这个拦截器内部重试或者重定向了 N 次，那么其内部嵌套的所有拦截器也会被调用N次，同样 networkInterceptors 自定义的拦截器也会被调用 N 次，如果直接从CacheInterceptors返回，那么networkInterceptors不会被调用到。而相对的 Interceptors 则一个请求只会调用一次，所以在OkHttp的内部也将其称之为 Application Interceptor。
