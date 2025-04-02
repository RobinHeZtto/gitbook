# OkHttp-connectionPool

### 1. 连接池的意义

* 降低资源消耗。频繁的进行建立Sokcet连接（TCP三次握手）和断开Socket（TCP四次分手）是非常消耗网络资源，将socket放到连接池复用能大大降低连接/断开网络的开销。
* 快速请求网络，低延时。请求时直接复用Socket连接，能大大提高网络访问的速度，HTTP中的keepalive连接对于降低延迟和提升速度有非常重要的作用。

### 2. 主要数据结构

#### ConnectionPool：

```
class ConnectionPool internal constructor(
  internal val delegate: RealConnectionPool
) {
  constructor(
    maxIdleConnections: Int,
    keepAliveDuration: Long,
    timeUnit: TimeUnit
  ) : this(RealConnectionPool(
      taskRunner = TaskRunner.INSTANCE,
      maxIdleConnections = maxIdleConnections,
      keepAliveDuration = keepAliveDuration,
      timeUnit = timeUnit
  ))

  constructor() : this(5, 5, TimeUnit.MINUTES)

  /** Returns the number of idle connections in the pool. */
  fun idleConnectionCount(): Int = delegate.idleConnectionCount()

  /** Returns total number of connections in the pool. */
  fun connectionCount(): Int = delegate.connectionCount()

  /** Close and remove all idle connections in the pool. */
  fun evictAll() {
    delegate.evictAll()
  }
}
```

* maxIdleConnections: 最大连接数，默认5个
* keepAliveDuration：连接最长保持时间5min
* RealConnectionPool：ConnectionPool的真正实现

#### RealConnectionPool：

```
class RealConnectionPool(
  taskRunner: TaskRunner,
  /** The maximum number of idle connections for each address. */
  private val maxIdleConnections: Int,
  keepAliveDuration: Long,
  timeUnit: TimeUnit
) {
  private val keepAliveDurationNs: Long = timeUnit.toNanos(keepAliveDuration)

  // 线程池清理任务
  private val cleanupQueue: TaskQueue = taskRunner.newQueue()
  private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
    override fun runOnce() = cleanup(System.nanoTime())
  }

  // 连接集合，基于链接节点的无界线程安全队列
  private val connections = ConcurrentLinkedQueue<RealConnection>()

  ......
}
```

**evictAll方法**

```
 //关闭所有请求
 fun evictAll() {
    //创建可变列表集合，用于搜集所有要关闭的请求对象
    val evictedConnections = mutableListOf<RealConnection>()
    synchronized(this) {
      val i = connections.iterator()
      //遍历连接池
      while (i.hasNext()) {
        val connection = i.next()
        //当所有的调用发射器为空，我们认为当前连接可以关闭
        if (connection.transmitters.isEmpty()) {
          connection.noNewExchanges = true
          evictedConnections.add(connection)
          i.remove()
        }
      }
    }

    for (connection in evictedConnections) {
      //关闭socket， 并忽略所有的可检查的Exception
      connection.socket().closeQuietly()
    }
}
```

**put方法：**

有新的连接时，把连接对象添加进连接池。

```
fun put(connection: RealConnection) {
   assert(Thread.holdsLock(this))
   //如果当前clieanupRunning设置为false，将清除线程添加进线程池。
   if (!cleanupRunning) {
     cleanupRunning = true
     executor.execute(cleanupRunnable)
   }
   //往队列里添加连接对象
   connections.add(connection)
}
```

**清除线程：**

当idle连接已经超过最大存活时间或最大连接时间，将idle连接清除出连接池，保持连接池的最佳性能。

```
private val cleanupRunnable = object : Runnable {
    override fun run() {
      while (true) {//开启循环
        val waitNanos = cleanup(System.nanoTime()) 
        if (waitNanos == -1L) return //当没有连接了，退出循环
        try {
          //与Java Object#wait的区别是， 当传入参数0的时候， 代表不再等待而不是Java wait的无限等待
          this@RealConnectionPool.lockAndWaitNanos(waitNanos) 
        } catch (_: InterruptedException) {
        }
      }
    }
```

```
  fun cleanup(now: Long): Long {
    var inUseConnectionCount = 0
    var idleConnectionCount = 0
    var longestIdleConnection: RealConnection? = null
    var longestIdleDurationNs = Long.MIN_VALUE

    // Find either a connection to evict, or the time that the next eviction is due.
    for (connection in connections) {
      synchronized(connection) {
        // If the connection is in use, keep searching.
        // 判断连接是否在使用
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++
        } else {
          idleConnectionCount++

          // If the connection is ready to be evicted, we're done.
          val idleDurationNs = now - connection.idleAtNs
          //当前存活最长的闲置连接以及其存活时间
          if (idleDurationNs > longestIdleDurationNs) {
            longestIdleDurationNs = idleDurationNs
            longestIdleConnection = connection
          } else {
            Unit
          }
        }
      }
    }

    when {
      longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections -> {
        // We've chosen a connection to evict. Confirm it's still okay to be evict, then close it.
        val connection = longestIdleConnection!!
        synchronized(connection) {
          if (connection.calls.isNotEmpty()) return 0L // No longer idle.
          if (connection.idleAtNs + longestIdleDurationNs != now) return 0L // No longer oldest.
          connection.noNewExchanges = true
          //闲置连接大于最大保持存活时间，清除出连接池
          connections.remove(longestIdleConnection)
        }

        connection.socket().closeQuietly()
        if (connections.isEmpty()) cleanupQueue.cancelAll()

        // Clean up again immediately.
        return 0L
      }

      idleConnectionCount > 0 -> {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs
      }

      inUseConnectionCount > 0 -> {
        // All connections are in use. It'll be at least the keep alive duration 'til we run
        // again.
        return keepAliveDurationNs
      }

      else -> {
        // No connections, idle or in use.
        return -1
      }
    }
  }

```

#### RealConnection：

RealConnection是Connection的实现类，代表着链接socket的链路，如果拥有一个RealConnection就代表已经跟服务器有了一条通信链路。

**主要成员：**

```
  /** The low-level TCP socket. */
  private var rawSocket: Socket? = null

  /**
   * The application layer socket. Either an [SSLSocket] layered over [rawSocket], or [rawSocket]
   * itself if this connection does not use SSL.
   */
  private var socket: Socket? = null
  // 握手信息
  private var handshake: Handshake? = null
  // 协议
  private var protocol: Protocol? = null
  // http2的链接
  private var http2Connection: Http2Connection? = null
  // 输入输出流
  private var source: BufferedSource? = null
  private var sink: BufferedSink? = null


  // 从连接池移除以后置为true，表示不能够再使用该连接创建stream
  var noNewExchanges = false
```

**Connect方法：**

```
public void connect(。。。) {
    ....

    while (true) {//一个while循环
         //如果是https请求并且使用了http代理服务器
        if (route.requiresTunnel()) {
          connectTunnel(...);
        } else {//
            //直接创建socket链接
          connectSocket(connectTimeout, readTimeout);
        }
        //建立协议
        establishProtocol(connectionSpecSelector);
        break;//跳出while循环
    ......
  }
```

**isEligible方法：**

```
internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
  assertThreadHoldsLock()

  // 连接是否超限，HTTP/1每个HTTP连接只能对应一个请求。
  // noNewExchanges连接是否还可用，否则返回false
  if (calls.size >= allocationLimit || noNewExchanges) return false

  // dns，代理，协议，端口，hostnameVerifier，certificatePinner等是否一致，不一致不能复用
  if (!this.route.address.equalsNonHost(address)) return false

  // host是否一样，如果一样可以复用
  if (address.url.host == this.route().address.url.host) {
    return true // This connection is a perfect match.
  }
  
  // 对于HTTP/2de连接，如果Host不一样，但是ip一样，端口，证书等一样，也可以复用
  // 1. This connection must be HTTP/2.
  if (http2Connection == null) return false

  // 2. The routes must share an IP address.
  if (routes == null || !routeMatchesAny(routes)) return false

  // 3. This connection's server certificate's must cover the new host.
  if (address.hostnameVerifier !== OkHostnameVerifier) return false
  if (!supportsUrl(address.url)) return false

  // 4. Certificate pinning must match the host.
  try {
    address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
  } catch (_: SSLPeerUnverifiedException) {
    return false
  }

  return true // The caller's address can be carried by this connection.
}
```

对失败的路由进行追踪， 并添加进一个黑名单里，失败黑名单的路由连接优先级将降到最低， 其它路由先做连接尝试。

```
internal fun connectFailed(client: OkHttpClient, failedRoute: Route, failure: IOException) {
  // Tell the proxy selector when we fail to connect on a fresh connection.
  if (failedRoute.proxy.type() != Proxy.Type.DIRECT) {
    val address = failedRoute.address
    address.proxySelector.connectFailed(
        address.url.toUri(), failedRoute.proxy.address(), failure)
  }

  client.routeDatabase.failed(failedRoute)
}
```

 



