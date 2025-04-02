# Android基础之Handler全解析

> 在Android中进程间通信主要采用的是Binder，线程间通信使用的是Handler机制。

### 1. 关键角色

Handler消息驱动机制主要包含以下关键角色：

* **Message**：传递的消息；
* **MessageQueue**：消息队列的主要功能向消息池投递消息(`MessageQueue.enqueueMessage`)和取走消息池的消息(`MessageQueue.next`)；
* **Handler**：消息辅助类，主要功能向消息池发送各种消息事件(`Handler.sendMessage`)和处理相应消息事件(`Handler.handleMessage`)；
* **Looper**：不断循环执行(`Looper.loop`)，按分发机制将消息分发给目标处理者。

![](<../../.gitbook/assets/image (52).png>)

* 每个线程最多可以有一个Looper，Looper对应一个MessageQueue消息队列；
* MessageQueue包含一组待处理的Message；
* Message通过target指向处理该消息的Handler；
* Handler与其所在线程的Looper和MessageQueue对应；

### 2. Looper

#### prepare() <a href="#id-21-prepare" id="id-21-prepare"></a>

对于无参的情况，默认调用`prepare(true)`，表示的是这个Looper允许退出，而对于false的情况则表示当前Looper不允许退出。一般我们自定义线程中，可以退出。但是在主线程中设置不可退出。

```
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

private static void prepare(boolean quitAllowed) {
    //每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //创建Looper对象，并保存到当前线程的TLS区域
    sThreadLocal.set(new Looper(quitAllowed));
}
```

这里的`sThreadLocal`是ThreadLocal类型，下面，先说说ThreadLocal。

**ThreadLocal**： 线程本地存储区（Thread Local Storage，简称为TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。TLS常用的操作方法：

* `ThreadLocal.set(T value)`：将value存储到当前线程的TLS区域，源码如下：

```
public void set(T value) {
    Thread currentThread = Thread.currentThread(); //获取当前线程
    Values values = values(currentThread); //查找当前线程的本地储存区
    if (values == null) {
        //当线程本地存储区，尚未存储该线程相关信息时，则创建Values对象
        values = initializeValues(currentThread);
    }
    //保存数据value到当前线程this
    values.put(this, value);
}
```

* `ThreadLocal.get()`：获取当前线程TLS区域的数据，源码如下：

```
public T get() {
    Thread currentThread = Thread.currentThread(); //获取当前线程
    Values values = values(currentThread); //查找当前线程的本地储存区
    if (values != null) {
        Object[] table = values.table;
        int index = hash & values.mask;
        if (this.reference == table[index]) {
            return (T) table[index + 1]; //返回当前线程储存区中的数据
        }
    } else {
        //创建Values对象
        values = initializeValues(currentThread);
    }
    return (T) values.getAfterMiss(this); //从目标线程存储区没有查询是则返回null
}
```

ThreadLocal的get()和set()方法操作的类型都是泛型，接着回到前面提到的`sThreadLocal`变量，其定义如下：

```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>()
```

可见`sThreadLocal`的get()和set()操作的类型都是`Looper`类型。

**Looper.prepare()**

Looper.prepare()在每个线程只允许执行一次，该方法会创建Looper对象，Looper的构造方法中会创建一个MessageQueue对象，再将Looper对象保存到当前线程TLS。

对于Looper类型的构造方法如下：

```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);  //创建MessageQueue对象.
    mThread = Thread.currentThread();  //记录当前线程.
}
```

另外，与prepare()相近功能的，还有一个`prepareMainLooper()`方法，该方法主要在ActivityThread类中使用。

```
public static void prepareMainLooper() {
    prepare(false); //设置不允许退出的Looper
    synchronized (Looper.class) {
        //将当前的Looper保存为主Looper，每个线程只允许执行一次。
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

#### loop() <a href="#id-22-loop" id="id-22-loop"></a>

```
public static void loop() {
    final Looper me = myLooper();  //获取TLS存储的Looper对象 
    final MessageQueue queue = me.mQueue;  //获取Looper对象中的消息队列

    Binder.clearCallingIdentity();
    //确保在权限检查时基于本地进程，而不是调用进程。
    final long ident = Binder.clearCallingIdentity();

    for (;;) { //进入loop的主循环方法
        Message msg = queue.next(); //可能会阻塞 
        if (msg == null) { //没有消息，则退出循环
            return;
        }

        //默认为null，可通过setMessageLogging()方法来指定输出，用于debug功能
        Printer logging = me.mLogging;  
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        //分发Message 
        msg.target.dispatchMessage(msg); 
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        //恢复调用者信息
        final long newIdent = Binder.clearCallingIdentity();
        //将Message放入消息池 
        msg.recycleUnchecked();  
    }
}
```

loop()进入循环模式，不断重复下面的操作，直到没有消息时退出循环

* 读取MessageQueue的下一条Message；
* 把Message分发给相应的target；
* 再把分发后的Message回收到消息池，以便重复利用。

这是这个消息处理的核心部分。另外，上面代码中可以看到有logging方法，这是用于debug的，默认情况下`logging == null`，通过设置setMessageLogging()用来开启debug工作。

#### quit() <a href="#id-23-quit" id="id-23-quit"></a>

```
public void quit() {
    //消息移除，移除所有消息
    mQueue.quit(false); 
}

public void quitSafely() {
    //安全地消息移除，只会清空MessageQueue消息池中所有的延迟消息
    mQueue.quit(true); 
}
```

Looper.quit()方法的实现最终调用的是MessageQueue.quit()方法

**MessageQueue.quit()**

```
void quit(boolean safe) {
        // 当mQuitAllowed为false，表示不运行退出，强行调用quit()会抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        synchronized (this) {
            if (mQuitting) { //防止多次执行退出操作
                return;
            }
            mQuitting = true;
            if (safe) {
                //移除尚未触发的所有消息
                removeAllFutureMessagesLocked(); 
            } else {
                //移除所有的消息
                removeAllMessagesLocked(); 
            }
            //mQuitting=false，那么认定为 mPtr != 0
            nativeWake(mPtr);
        }
    }
```

消息退出的方式：

* 当safe =true时，只移除尚未触发的所有消息，对于正在触发的消息并不移除；
* 当safe =flase时，移除所有的消息

**myLooper**

用于获取TLS存储的Looper对象

```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

**post**

发送消息，并设置消息的callback，用于处理消息。

```
public final boolean post(Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

### 3. Handler <a href="#san-handler" id="san-handler"></a>

#### 3.1 创建Handler <a href="#id-31-chuang-jian-handler" id="id-31-chuang-jian-handler"></a>

**无参构造（被弃用，必须指定Looper）**

```
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    //匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
    mLooper = Looper.myLooper();  //从当前线程的TLS中获取Looper对象
    if (mLooper == null) {
        throw new RuntimeException("");
    }
    mQueue = mLooper.mQueue; //消息队列，来自Looper对象
    mCallback = callback;  //回调方法
    mAsynchronous = async; //设置消息是否为异步处理方式
}
```

对于Handler的无参构造方法，默认采用当前线程TLS中的Looper对象，并且callback回调方法为null，且消息为同步处理方式。只要执行的Looper.prepare()方法，那么便可以获取有效的Looper对象。

**有参构造**

```
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

Handler类在构造方法中，可指定Looper，Callback回调方法以及消息的处理方式(同步或异步)，对于无参的handler，默认是当前线程的Looper。

#### 消息分发机制 <a href="#id-32-xiao-xi-fen-fa-ji-zhi" id="id-32-xiao-xi-fen-fa-ji-zhi"></a>

在Looper.loop()中，当发现有消息时，调用消息的目标handler，执行dispatchMessage()方法来分发消息。

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //当Message存在回调方法，回调msg.callback.run()方法；
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //当Handler存在Callback成员变量时，回调方法handleMessage()；
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //Handler自身的回调方法handleMessage()
        handleMessage(msg);
    }
}
```

**分发消息流程：**

1. 当`Message`的回调方法不为空时，则回调方法`msg.callback.run()`，其中callBack数据类型为Runnable,否则进入步骤2；
2. 当`Handler`的`mCallback`成员变量不为空时，则回调方法`mCallback.handleMessage(msg)`,否则进入步骤3；
3. 调用`Handler`自身的回调方法`handleMessage()`，该方法默认为空，Handler子类通过覆写该方法来完成具体的逻辑。

对于很多情况下，消息分发后的处理方法是第3种情况，即Handler.handleMessage()，一般地往往通过覆写该方法从而实现自己的业务逻辑。

#### 消息发送 <a href="#id-33-xiao-xi-fa-song" id="id-33-xiao-xi-fa-song"></a>

发送消息调用链：

![java\_sendmessage](http://gityuan.com/images/handler/java_sendmessage.png)

从上图，可以发现所有的发消息方式，最终都是调用`MessageQueue.enqueueMessage()`;

&#x20;**sendEmptyMessage**

```
public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
}
```

**sendEmptyMessageDelayed**

```
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
```

**sendMessageDelayed**

```
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

**sendMessageAtTime**

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

**sendMessageAtFrontOfQueue**

```
public final boolean sendMessageAtFrontOfQueue(Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}
```

该方法通过设置消息的触发时间为0，从而使Message加入到消息队列的队头。

**post**

```
public final boolean post(Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

**postAtFrontOfQueue**

```
public final boolean postAtFrontOfQueue(Runnable r) {
    return sendMessageAtFrontOfQueue(getPostMessage(r));
}
```

**enqueueMessage**

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis); 
}
```

`Handler.sendEmptyMessage()`等系列方法最终调用`MessageQueue.enqueueMessage(msg, uptimeMillis)`，将消息添加到消息队列中，其中uptimeMillis为系统当前的运行时间，不包括休眠时间。

#### 3.4 Handler其他方法 <a href="#id-34handler-qi-ta-fang-fa" id="id-34handler-qi-ta-fang-fa"></a>

**obtainMessage**

获取消息

```
public final Message obtainMessage() {
    return Message.obtain(this); 
}
```

`Handler.obtainMessage()`方法，最终调用`Message.obtainMessage(this)`，其中this为当前的Handler对象。

**removeMessages**

```
public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null); 
}
```

`Handler`是消息机制中非常重要的辅助类，更多的实现都是`MessageQueue`, `Message`中的方法，Handler的目的是为了更加方便的使用消息机制。

### 4. MessageQueue

MessageQueue消息入队过程：

```
boolean enqueueMessage(Message msg, long when) {
    // 除了同步屏障，其他消息的target必须不为null
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        // 正在使用的消息，不能再次使用，直接抛出异常
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        // 消息队列正在退出时，不再入队了，进行msg回收，加入到消息池
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        // 标记消息正在使用
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        // p为null(代表MessageQueue没有消息） 
        // 或者msg的触发时间是队列中最早的， 加入到队头
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            // 如果当前没有在唤醒状态（取消息执行中），mBlocked为true
            needWake = mBlocked;
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
            //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

`MessageQueue`是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

MessageQueue获取下一条消息过程：

```
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    //当消息循环已经退出，则直接返回
    if (ptr == 0) {
        return null;
    }
    // 循环迭代的首次为-1
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            // 如果对头的message为消息屏障，即当msg.target为空时，
            // 需要先处理队列内的异步消息
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                //循环查询异步消息，查询到则退出循环
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                //当消息触发时间大于当前时间，则设置下一次轮询的超时时长
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    // 返回到Looper.loop中进行消息分发
                    return msg;
                }
            } else {
                //没有消息
                nextPollTimeoutMillis = -1;
            }

            // 消息队列正在退出，不再返回任何消息
            if (mQuitting) {
                dispose();
                return null;
            }

            //当消息队列为空，或者是消息队列的第一个消息时
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //没有idle handlers需要运行，继续循环，再进入nativePollOnce等待
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 只有第一次循环时，会运行idle handlers，执行完成后，
        // 重置pendingIdleHandlerCount为0.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        //重置idle handler个数为0，以保证不会再次重复运行
        pendingIdleHandlerCount = 0;

        //当调用一个空闲handler时，一个新message能够被分发，
        //因此无需等待可以直接查询pending message.
        nextPollTimeoutMillis = 0;
    }
}
```

`nativePollOnce`是阻塞操作，其中`nextPollTimeoutMillis`代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。

当处于空闲时，往往会执行`IdleHandler`中的方法。当nativePollOnce()返回后，next()从`mMessages`中提取一个消息。

堆栈信息：

```
  native: #00 pc 00019178  /system/lib/libc.so (syscall+28)
  native: #01 pc 000b3e35  /system/lib/libart.so (_ZN3art17ConditionVariable16WaitHoldingLocksEPNS_6ThreadE+88)
  native: #02 pc 0026302f  /system/lib/libart.so (_ZN3art3JNI16CallObjectMethodEP7_JNIEnvP8_jobjectP10_jmethodIDz+358)
  native: #03 pc 00002b57  /system/lib/libnativehelper.so (jniGetReferent+90)
  native: #04 pc 000c8b09  /system/lib/libandroid_runtime.so (_ZN7android26NativeDisplayEventReceiver13dispatchVsyncExij+20)
  native: #05 pc 00032555  /system/lib/libandroidfw.so (_ZN7android22DisplayEventDispatcher11handleEventEiiPv+104)
  native: #06 pc 000105cd  /system/lib/libutils.so (_ZN7android6Looper9pollInnerEi+572)
  native: #07 pc 000102f9  /system/lib/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+32)
  native: #08 pc 000e47dd  /system/lib/libandroid_runtime.so (_ZN7android18NativeMessageQueue8pollOnceEP7_JNIEnvP8_jobjecti+24)
  native: #09 pc 001a8df5  /system/framework/arm/boot-framework.oat (Java_android_os_MessageQueue_nativePollOnce__JI+92)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:325)
  at android.os.Looper.loop(Looper.java:142)
  at android.app.ActivityThread.main(ActivityThread.java:6938)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:327)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1374)
```

### 5. Native层实现

在Native层，也有创建MessageQueue及Looper对象，入口：

```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

整套消息队列的唤醒机制是通过epoll来实现的，具体可以参考 [Android消息机制2-Handler(Native层)](http://gityuan.com/2015/12/27/handler-message-native/)

### 6. 消息屏障

在Android的消息机制中，Message其实有分三种消息: **普通消息、异步消息及同步屏障**。

* 普通消息：通过handler普通发送的消息
* 异步消息：设置message FLAG\_ASYNCHRONOUS标志的消息
* 同步屏障：message的target为空的消息

同步屏障是为了保证异步消息优先执行的机制，一般使用方式为，先postSyncBarrier同步屏障消息，然后再发送异步消息，同步屏障消息之后的异步消息会优先处理，直到全部处理完成才会处理其他消息。

```
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        // 没有设置target
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        // 根据触发时间插到队列对应的位置
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

同步屏障消息通过token来查找，移除时传入的是token。

```
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

#### 如何发送异步消息：

```
public Handler(boolean async);
public Handler(Callback callback, boolean async);
public Handler(Looper looper, Callback callback, boolean async);

Message.setAsynchronous(true);
```

#### 同步屏障的应用场景

&#x20;Android框架中为了更快的响应UI刷新绘制事件，在`ViewRootImpl.scheduleTraversals`中使用了同步屏障。

```
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //设置同步障碍，确保mTraversalRunnable优先被执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //内部通过Handler发送了一个异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}


```

### 7. IdleHandler

IdleHandler时消息队列空闲执行的机制，一般用于低优先级任务。在MessageQueue.next中，循环取完当前需要处理的message之后，没有消息需要处理时，会调用到IdleHandler。

```
public final class MessageQueue {
    ......
    /**
     * 当前队列将进入阻塞等待消息时调用该接口回调，即队列空闲
     */
    public static interface IdleHandler {
        /**
         * 返回true就是单次回调后不删除，下次进入空闲时继续回调该方法，false只回调单次。
         */
        boolean queueIdle();
    }

    /**
     * 判断当前队列是不是空闲的，辅助方法
     */
    public boolean isIdle() {
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            return mMessages == null || now < mMessages.when;
        }
    }

    /**
     * 添加一个IdleHandler到队列，如果IdleHandler接口方法返回false则执行完会自动删除，
     * 否则需要手动removeIdleHandler。
     */
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
    
    /**
     * 删除一个之前添加的 IdleHandler。
     */
    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }
    
    Message next() {
    ......
    for (;;) {
        ......
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            ......
            //把通过addIdleHandler添加的IdleHandler转成数组存起来在mPendingIdleHandlers中
            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        //循环遍历所有IdleHandler
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                //调用IdleHandler接口的queueIdle方法并获取返回值。
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            //如果IdleHandler接口的queueIdle方法返回false说明只执行一次需要删除。
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        ......
    }
    
}
```

#### 使用方式：

```
Handler(Looper.myLooper()!!).looper.queue.addIdleHandler { 
    // doSomething
    true
}
```

返回true表示下次idle时还需要执行，false表示下次不需要执行。
