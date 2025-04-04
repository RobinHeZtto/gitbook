# OkHttp-dispatcher

&#x20;       **Okhttp中的Dispatcher用来调度分发同步/异步请求。**&#x6211;们可以通过 OkHtpClient#dispatcher() 获取 Dispatcher 对象对请求进行统一的控制，例如结束所有请求、获取线程池等等。

### 1. Dispatcher工作原理

&#x20;       Dispatcher 中维护三个队列：

* readyAsyncCalls：异步请求就绪队列，一个新的异步请求首先会被加入该队列中。
* runningAsyncCalls：当前运行的异步请求队列
* runningSyncCalls：当前运行的同步请求队列

&#x20;       Dispatcher 中还包含一个默认的线程池用于执行所有的异步请求，也可以通过构造器指定一个线程池，所有的异步请求都将会通过这个线程池来执行。

```
// 配置线程池
val executorService = ThreadPoolExecutor(0, Int.MAX_VALUE, 120,
    TimeUnit.MILLISECONDS, SynchronousQueue<Runnable>())
val dispatcher = Dispatcher(executorService)
val okHttpClient = OkHttpClient().newBuilder().dispatcher(dispatcher).build()

// 配置请求数
okHttpClient.dispatcher.maxRequests = 128
okHttpClient.dispatcher.maxRequestsPerHost = 10

// 配置空闲回调方法
okHttpClient.dispatcher.idleCallback = Runnable {  }
```

         **OkHttp dispatcher调度流程：**&#x4E3B;要实现在promoteAndExecute方法中。当加入**异步请求**执行时，如果当前正在运行的请求数大于maxRequest或者同一域名主机请求数大于maxRequestsPerHost，直接放到readyAsyncCalls等待执行，否则将放入runningAsyncCalls队列执行。

![](<../../../.gitbook/assets/image (293).png>)

&#x20;       **调度时机：新请求，请求执行完成，重新设置请求数maxRequest或maxRequestsPerHost限制**

```
/*
 * Copyright (C) 2013 Square, Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package okhttp3

import java.util.ArrayDeque
import java.util.Collections
import java.util.Deque
import java.util.concurrent.ExecutorService
import java.util.concurrent.SynchronousQueue
import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit
import okhttp3.internal.assertThreadDoesntHoldLock
import okhttp3.internal.connection.RealCall
import okhttp3.internal.connection.RealCall.AsyncCall
import okhttp3.internal.okHttpName
import okhttp3.internal.threadFactory

/**
 * 负责异步请求的分发调度
 * 可以自定义配置executorService(默认使用CachedThreadPool), maxRequests（默认64）,
 * maxRequestsPerHost（默认5）
 */
class Dispatcher constructor() {
  // 最大并发请求数
  @get:Synchronized var maxRequests = 64
    set(maxRequests) {
      require(maxRequests >= 1) { "max < 1: $maxRequests" }
      synchronized(this) {
        field = maxRequests
      }
      promoteAndExecute()
    }

  // 每个host的最大并发请求数。并非ip限制，可能会有多个host对应同一个ip或者代理的情况，
  // 这个时候，对单个IP的访问就会超出此限制。
  @get:Synchronized var maxRequestsPerHost = 5
    set(maxRequestsPerHost) {
      require(maxRequestsPerHost >= 1) { "max < 1: $maxRequestsPerHost" }
      synchronized(this) {
        field = maxRequestsPerHost
      }
      promoteAndExecute()
    }

  // dispatcher变空闲时的回调，在所有异步或者同步操作执行完成后回掉
  @set:Synchronized
  @get:Synchronized
  var idleCallback: Runnable? = null

  // 线程池
  private var executorServiceOrNull: ExecutorService? = null

  // 在没有指定的情况下默认使用CachedThreadPool
  @get:Synchronized
  @get:JvmName("executorService") val executorService: ExecutorService
    get() {
      if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
            SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
      }
      return executorServiceOrNull!!
    }

  // 等待执行的异步请求
  private val readyAsyncCalls = ArrayDeque<AsyncCall>()

  // 正在执行中的异步请求
  private val runningAsyncCalls = ArrayDeque<AsyncCall>()

  // 正在执行中的同步请求
  private val runningSyncCalls = ArrayDeque<RealCall>()

  constructor(executorService: ExecutorService) : this() {
    this.executorServiceOrNull = executorService
  }

  // 异步请求
  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      readyAsyncCalls.add(call)

      // 计算同一host上的请求数，通过AtomicInteger变量计数
      if (!call.call.forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host)
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    // 执行请求
    promoteAndExecute()
  }

  // 在runningAsyncCalls与readyAsyncCalls队列上查找对应host的AsyncCall
  private fun findExistingCallWithHost(host: String): AsyncCall? {
    for (existingCall in runningAsyncCalls) {
      if (existingCall.host == host) return existingCall
    }
    for (existingCall in readyAsyncCalls) {
      if (existingCall.host == host) return existingCall
    }
    return null
  }

  // 取消所有同步/异步请求队列，包括正在请求中的
  @Synchronized fun cancelAll() {
    for (call in readyAsyncCalls) {
      call.call.cancel()
    }
    for (call in runningAsyncCalls) {
      call.call.cancel()
    }
    for (call in runningSyncCalls) {
      call.cancel()
    }
  }
  
  // 将readyAsyncCalls中的调度到runningAsyncCalls中并执行，返回是否有请求正在运行
  private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      // 遍历readyAsyncCalls队列
      val i = readyAsyncCalls.iterator()
      while (i.hasNext()) {
        val asyncCall = i.next()
        // 当前执行的请求超过设定的最大请求数，直接退出
        if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
        // 待执行请求的host超过了并发请求上限，不能执行，继续在readyAsyncCalls中等待
        if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

        // 从readyAsyncCalls中移除，并加入到runningAsyncCalls中
        i.remove()
        asyncCall.callsPerHost.incrementAndGet()
        executableCalls.add(asyncCall)
        runningAsyncCalls.add(asyncCall)
      }
      isRunning = runningCallsCount() > 0
    }

    // 在线程池中运行加入到runningAsyncCalls队列中的新请求
    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService)
    }

    return isRunning
  }

  // 运行同步请求，仅加入到runningSyncCalls队列，执行还是在主线程中
  @Synchronized internal fun executed(call: RealCall) {
    runningSyncCalls.add(call)
  }

  // 异步请求完成
  internal fun finished(call: AsyncCall) {
    call.callsPerHost.decrementAndGet()
    finished(runningAsyncCalls, call)
  }

  // 同步请求完成
  internal fun finished(call: RealCall) {
    finished(runningSyncCalls, call)
  }

  private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
    synchronized(this) {
      // 从队列中移除请求完成的call
      if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
      idleCallback = this.idleCallback
    }

    // 继续调度二个队列的请求
    val isRunning = promoteAndExecute()

    // 如果当前没有请求在进行，且idleCallback不为空，回掉idleCallback
    if (!isRunning && idleCallback != null) {
      idleCallback.run()
    }
  }

  /** Returns a snapshot of the calls currently awaiting execution. */
  @Synchronized fun queuedCalls(): List<Call> {
    return Collections.unmodifiableList(readyAsyncCalls.map { it.call })
  }

  /** Returns a snapshot of the calls currently being executed. */
  @Synchronized fun runningCalls(): List<Call> {
    return Collections.unmodifiableList(runningSyncCalls + runningAsyncCalls.map { it.call })
  }

  @Synchronized fun queuedCallsCount(): Int = readyAsyncCalls.size

  @Synchronized fun runningCallsCount(): Int = runningAsyncCalls.size + runningSyncCalls.size

  @JvmName("-deprecated_executorService")
  @Deprecated(
      message = "moved to val",
      replaceWith = ReplaceWith(expression = "executorService"),
      level = DeprecationLevel.ERROR)
  fun executorService(): ExecutorService = executorService
}
```

### 2. Dispatcher 线程池

&#x20;       dispatcher默认使用的是缓存线程池CachedThreadPool，请求无等待，实现最大并发。

```
@get:Synchronized
@get:JvmName("executorService") val executorService: ExecutorService
  get() {
    if (executorServiceOrNull == null) {
      executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
          SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
    }
    return executorServiceOrNull!!
  }
```

&#x20;     dispatcher的线程池定义如上，其实与`Executors.newCachedThreadPool()`一样。首先核心线程为0，表示线程池不会一直为我们缓存线程，线程池中所有线程都是在60s内没有工作就会被回收。而最大线程`Integer.MAX_VALUE`与等待队列`SynchronousQueue`的组合能够得到最大的吞吐量。即当需要线程池执行任务时，如果不存在空闲线程不需要等待，马上新建线程执行任务！等待队列的不同指定了线程池的不同排队机制。一般来说，等待队列`BlockingQueue`有：`ArrayBlockingQueue`、`LinkedBlockingQueue`与`SynchronousQueue`。

&#x20;       假设向线程池提交任务时，核心线程都被占用的情况下：

**`ArrayBlockingQueue`**：基于数组的阻塞队列，初始化需要指定固定大小。

&#x20;        当使用此队列时，向线程池提交任务，会首先加入到等待队列中，当等待队列满了之后，再次提交任务，尝试加入队列就会失败，这时就会检查如果当前线程池中的线程数未达到最大线程，则会新建线程执行新提交的任务。所以最终可能出现后提交的任务先执行，而先提交的任务一直在等待。

**`LinkedBlockingQueue`**：基于链表实现的阻塞队列，初始化可以指定大小，也可以不指定。

&#x20;        当指定大小后，行为就和`ArrayBlockingQueu`一致。**而如果未指定大小，则会使用默认的`Integer.MAX_VALUE`作为队列大小**。这时候就会出现线程池的最大线程数参数无用，因为无论如何，向线程池提交任务加入等待队列都会成功。最终意味着所有任务都是在核心线程执行。如果核心线程一直被占，那就一直等待。

**`SynchronousQueue`** : 无容量的队列。

&#x20;        使用此队列意味着希望获得最大并发量。因为无论如何，向线程池提交任务，往队列提交任务都会失败。而失败后如果没有空闲的非核心线程，就会检查如果当前线程池中的线程数未达到最大线程，则会新建线程执行新提交的任务。完全没有任何等待，唯一制约它的就是最大线程数的个数。因此**一般配合`Integer.MAX_VALUE`就实现了真正的无等待**。

&#x20;       需要注意的是，系统资源是存在限制的，而每一个线程都需要分配一定的系统资源，所以线程并不能无限个数。那么当设置最大线程数为`Integer.MAX_VALUE`时，**OkHttp同时还有最大请求任务执行个数: 64**的限制，这样即解决了这个问题同时也能获得最大吞吐。
