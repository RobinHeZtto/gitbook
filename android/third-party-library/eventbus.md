# EventBus

### 1. 事件总线

事件总线是对发布-订阅模式的一种实现。它是一种集中式事件处理机制，允许不同的组件之间进行彼此通信而又不需要相互依赖，达到解耦的目的。

![](<../../.gitbook/assets/image (317).png>)

发布订阅模式主要有两个角色：

* **发布方（Publisher）**：也称为被观察者，当状态改变时负责通知所有订阅者。
* **订阅方（Subscriber）**：也称为观察者，订阅事件并对接收到的事件进行处理。

发布订阅模式有两种实现方式：

* 简单的实现方式：由Publisher维护一个订阅者列表，当状态改变时循环遍历列表通知订阅者。
* 委托的实现方式：由Publisher定义事件委托，Subscriber实现委托。

总的来说，发布订阅模式二个核心点，通知与更新。被观察者状态改变通知观察者做出相应更新，\
解决的是当对象改变时需要通知其他对象做出相应改变的问题。

**为什么要使用事件总线？**

EventBus没出现之前，一般直接使用广播进行组件间的消息传递，传统的方法实现，往往代码不够优雅，使用方式复杂。

事件总线的优缺点：

* 优点：开销小，代码更优雅、简洁，解耦发送者和接收者，可动态设置事件处理线程和优先级。
* 缺点：每个事件必须自定义一个事件类，增加了维护成本。

### 2. 使用方式

1、定义事件

```
public class MyEvent { ... }
```

2、订阅，声明订阅方法

```
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessageEvent(MyEvent event) {
    
}
```

3、注册、反注册

```
@Override
public void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}

@Override
public void onStop() {
    super.onStop();
    EventBus.getDefault().unregister(this);
}
```

4、发送事件

```
EventBus.getDefault().post(new MyEvent());
```

### 3. 源码解析

#### 关键类：

Subscription 订阅者

```
final class Subscription {
　　 // 订阅的对象
    final Object subscriber;
    // 订阅对象上的具体方法
    final SubscriberMethod subscriberMethod;
    // 取消订阅以后，该值为flase
    volatile boolean active;

    Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }

    @Override
    public boolean equals(Object other) {
        if (other instanceof Subscription) {
            Subscription otherSubscription = (Subscription) other;
            return subscriber == otherSubscription.subscriber
                    && subscriberMethod.equals(otherSubscription.subscriberMethod);
        } else {
            return false;
        }
    }

    @Override
    public int hashCode() {
        return subscriber.hashCode() + subscriberMethod.methodString.hashCode();
    }
}
```

SubscriberMethod 订阅对象上的方法

```
public class SubscriberMethod {
    // 方法
    final Method method;
    // 执行模式
    final ThreadMode threadMode;
    // 事件类型
    final Class<?> eventType;
    // 优先级
    final int priority;
    // 是否粘性
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }

    @Override
    public boolean equals(Object other) {
        if (other == this) {
            return true;
        } else if (other instanceof SubscriberMethod) {
            checkMethodString();
            SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
            otherSubscriberMethod.checkMethodString();
            // Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }

    private synchronized void checkMethodString() {
        if (methodString == null) {
            // Method.toString has more overhead, just take relevant parts of the method
            StringBuilder builder = new StringBuilder(64);
            builder.append(method.getDeclaringClass().getName());
            builder.append('#').append(method.getName());
            builder.append('(').append(eventType.getName());
            methodString = builder.toString();
        }
    }

    @Override
    public int hashCode() {
        return method.hashCode();
    }
}
```

#### EventBus.getDefault().register(this) <a href="#er-eventbusgetdefaultregisterthis" id="er-eventbusgetdefaultregisterthis"></a>

首先，我们从获取EventBus实例的方法getDefault()开始分析：

```
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```

在getDefault()中使用了DCL的单例模式来创建EventBus实例。

```
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

public EventBus() {
    this(DEFAULT_BUILDER);
}
```

EventBus参构造方法：

```
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
private final Map<Object, List<Class<?>>> typesBySubscriber;
private final Map<Class<?>, Object> stickyEvents;

EventBus(EventBusBuilder builder) {
    ...

    // 1
    subscriptionsByEventType = new HashMap<>();

    // 2
    typesBySubscriber = new HashMap<>();

    // 3
    stickyEvents = new ConcurrentHashMap<>();

    // 4
    mainThreadSupport = builder.getMainThreadSupport();
    mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);

    ...

    // 5
    subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
            builder.strictMethodVerification, builder.ignoreGeneratedIndex);

    // 从builder取中一些列订阅相关信息进行赋值
    ...

    // 6
    executorService = builder.executorService;
}
```

**注释1:** 创建subscriptionsByEventType对象，它是一个**以Event为key，Subscription链表为value的hashMap**。这里的Subscription是一个订阅信息对象，它里面保存了两个重要的字段，一个是类型为 Object 的 subscriber，该字段即为注册的对象（在 Android 中时通常是 Activity对象）；另一个是类型为SubscriberMethod 的 subscriberMethod，它就是被@Subscribe注解的那个订阅方法，里面保存了一个重要的字段eventType，它是 Class\<?> 类型的，代表了 Event 的类型。

**注释2:** 创建类型为 Map 的typesBySubscriber对象，它是一个**以subscriber为key，以subscriber对象为value的链表**，日常使用中仅用于判断某个对象是否注册过。

**注释3:**  新建了一个类型为ConcurrentHashMap的stickyEvents对象，它是专用于粘性事件处理的一个字段，**key为事件的Class对象，value为当前的事件**。

> 普通事件是先注册，然后发送事件才能收到；而粘性事件，在发送事件之后再订阅该事件也能收到。并且，粘性事件会保存在内存中，每次进入都会去内存中查找获取最新的粘性事件，除非你手动解除注册。

**注释4:**  新建三个不同类型的事件发送器

* mainThreadPoster：主线程事件发送器，通过它的mainThreadPoster.enqueue(subscription, event)方法可以将订阅信息和对应的事件进行入队，然后通过 handler 去发送一个消息，在 handler 的 handleMessage 中去执行方法。
* backgroundPoster：后台事件发送器，通过它的enqueue() 将方法加入到后台的一个队列，最后通过线程池去执行，注意，它在 Executor的execute()方法 上添加了 synchronized关键字 并设立 了控制标记flag，保证任一时间只且仅能有一个任务会被线程池执行。
* asyncPoster：实现逻辑类似于backgroundPoster，不同于backgroundPoster的保证任一时间只且仅能有一个任务会被线程池执行的特性，asyncPoster则是异步运行的，可以同时接收多个任务。

**注释5:**  新建一个subscriberMethodFinder对象，这是从EventBus中抽离出的订阅方法查询的一个对象

**注释6:** 从builder中取出了一个默认的线程池对象，它由Executors的newCachedThreadPool()方法创建，它是一个有则用、无则创建、无数量上限的线程池。

继续看EventBus的register()方法：

```
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();

    // 1
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // 2
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

**注释1:**  根据当前注册类获取 subscriberMethods这个订阅方法列表 。

**注释2:**  循环订阅SubscriberMethod。

SubscriberMethodFinder.findSubscriberMethods()获取订阅方法执行流程：

```
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 1
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }

    // 2
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

**注释1:**  首先从缓存METHOD\_CACHE中找，找到则直接返回。

**注释2:**  ignoreGeneratedIndex字段， 用来判断是否使用生成的 APT 代码生成。如果我们没有在项目中接入 EventBus 的 APT，那么可以将 ignoreGeneratedIndex 字段设为 false 以提高性能。这里ignoreGeneratedIndex 默认为false，所以会执行findUsingInfo()方法，后面生成 subscriberMethods 成功的话会加入到缓存中，失败的话会 抛出异常。

接着我们查看SubscriberMethodFinder的findUsingInfo()方法：

```
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    // 1
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    // 2
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod: array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
             // 3
             findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    // 4
    return getMethodsAndRelease(findState);
}
```

在注释1处，调用了SubscriberMethodFinder的prepareFindState()方法创建了一个新的 FindState 类，我们来看看这个方法：

```
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
private FindState prepareFindState() {
    // 1
    synchronized(FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    // 2
    return new FindState();
}
```

在注释1处，会先从 FIND\_STATE\_POOL 即 FindState 池中取出可用的 FindState（这里的POOL\_SIZE为4），如果没有的话，则通过注释2处的代码直接新建 一个新的 FindState 对象。

接着我们来分析下FindState这个类：

```
static class FindState {
    ....
    void initForSubscriber(Class<?> subscriberClass) {
        this.subscriberClass = clazz = subscriberClass;
        skipSuperClasses = false;
        subscriberInfo = null;
    }
    ...
}
```

它是 SubscriberMethodFinder 的内部类，这个方法主要做一个初始化、回收对象等工作。

我们接着回到SubscriberMethodFinder的注释2处的SubscriberMethodFinder()方法：

```
private SubscriberInfo getSubscriberInfo(FindState findState) {
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    if (subscriberInfoIndexes != null) {
        for (SubscriberInfoIndex index: subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    return null;
}
```

在前面初始化的时候，findState的subscriberInfo和subscriberInfoIndexes 这两个字段为空，所以这里直接返回 null。

接着我们查看注释3处的findUsingReflectionInSingleClass()方法：

```
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    for (Method method: methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?> [] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // 重点
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode, subscribeAnnotation.priority(),  subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification &&     method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException("@Subscribe method " + methodName + "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName + " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

这个方法很长，大概做的事情是：

* 1、通过反射的方式获取订阅者类中的所有声明方法，然后在这些方法里面寻找以 @Subscribe作为注解的方法进行处理。
* 2、在经过经过一轮检查，看看 findState.subscriberMethods是否存在，如果没有，将方法名，threadMode，优先级，是否为 sticky 方法等信息封装到 SubscriberMethod 对象中，最后添加到 subscriberMethods 列表中。

最后，我们继续查看注释4处的getMethodsAndRelease()方法：

```
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    // 1
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    // 2
    findState.recycle();
    // 3
    synchronized(FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    // 4
    return subscriberMethods;
}
```

在这里，首先在注释1处，从findState中取出了保存的subscriberMethods。在注释2处，将findState里的保存的所有对象进行回收。在注释3处，把findState存储在 FindState 池中方便下一次使用，以提高性能。最后，在注释4处，返回subscriberMethods。接着，在EventBus的 register() 方法的最后会调用 subscribe 方法：

```
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

我们继续看看这个subscribe()方法做的事情：

```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

    // 1
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList <> ();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
        }
    }
    int size = subscriptions.size();

    // 2
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    // 3
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    // 4
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if(eventType.isAssignableFrom(candidateEventType)) {
                Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

首先，在注释1处，会根据 subscriberMethod的eventType，在 subscriptionsByEventType 去查找一个 CopyOnWriteArrayList ，如果没有则创建一个新的 CopyOnWriteArrayList，然后将这个 CopyOnWriteArrayList 放入 subscriptionsByEventType 中。在注释2处，添加 newSubscription对象，它是一个 Subscription 类，里面包含着 subscriber 和 subscriberMethod 等信息，并且这里有一个优先级的判断，说明它是按照优先级添加的。优先级越高，会插到在当前 List 靠前面的位置。在注释3处，对typesBySubscriber 进行添加，这主要是在EventBus的isRegister()方法中去使用的，目的是用来判断这个 Subscriber对象 是否已被注册过。最后，在注释4处，会判断是否是 sticky事件。如果是sticky事件的话，会调用 checkPostStickyEventToSubscription() 方法。

我们接着查看这个checkPostStickyEventToSubscription()方法：

```
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

可以看到最终是调用了postToSubscription()这个方法来进行粘性事件的发送，对于粘性事件的处理，我们最后再分析，接下来我们看看事件是如何post的。

#### 三、EventBus.getDefault().post(new CollectEvent()) <a href="#san-eventbusgetdefaultpostnewcollectevent" id="san-eventbusgetdefaultpostnewcollectevent"></a>

```
public void post(Object event) {
    // 1
    PostingThreadState postingState = currentPostingThreadState.get();
    List <Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    // 2
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

注释1处，这里的currentPostingThreadState 是一个 ThreadLocal 类型的对象，里面存储了 PostingThreadState，而 PostingThreadState 中包含了一个 eventQueue 和其他一些标志位，相关的源码如下：

```
private final ThreadLocal <PostingThreadState> currentPostingThreadState = new ThreadLocal <PostingThreadState> () {
@Override
protected PostingThreadState initialValue() {
    return new PostingThreadState();
}
};

final static class PostingThreadState {
    final List <Object> eventQueue = new ArrayList<>();
    boolean isPosting;
    boolean isMainThread;
    Subscription subscription;
    Object event;
    boolean canceled;
}
```

接着把传入的 event，保存到了当前线程中的一个变量 PostingThreadState 的 eventQueue 中。在注释2处，最后调用了 postSingleEvent() 方法，我们继续查看这个方法：

```
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    // 1
    if (eventInheritance) {
        // 2
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |=
            // 3
            postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        ...
    }
}
```

首先，在注释1处，首先取出 Event 的 class 类型，接着会对 eventInheritance 标志位 判断，它默认为true，如果设为 true 的话，它会在发射事件的时候判断是否需要发射父类事件，设为 false，能够提高一些性能。接着，在注释2处，会调用lookupAllEventTypes() 方法，它的作用就是取出 Event 及其父类和接口的 class 列表，当然重复取的话会影响性能，所以它也做了一个 eventTypesCache 的缓存，这样就不用重复调用 getSuperclass() 方法。最后，在注释3处会调用postSingleEventForEventType()方法，我们看下这个方法：

```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class <?> eventClass) {
    CopyOnWriteArrayList <Subscription> subscriptions;
    synchronized(this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription: subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

可以看到，这里直接根据 Event 类型从 subscriptionsByEventType 中取出对应的 subscriptions对象，最后调用了 postToSubscription() 方法。

这个时候我们在看看这个postToSubscription()方法：

```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknow thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

从上面可以看出，这里通过threadMode 来判断在哪个线程中去执行方法：

* 1、POSTING：执行 invokeSubscriber() 方法，内部直接采用反射调用。
* 2、MAIN：首先去判断当前是否在 UI 线程，如果是的话则直接反射调用，否则调用mainThreadPoster的enqueue()方法，即把当前的方法加入到队列之中，然后通过 handler 去发送一个消息，在 handler 的 handleMessage 中去执行方法。
* 3、MAIN\_ORDERED：与MAIN类型，不过是确保是顺序执行的。
* 4、BACKGROUND：判断当前是否在 UI 线程，如果不是的话直接反射调用，是的话通过backgroundPoster的enqueue()方法 将方法加入到后台的一个队列，最后通过线程池去执行。注意，backgroundPoster在 Executor的execute()方法 上添加了 synchronized关键字 并设立 了控制标记flag，保证任一时间只且仅能有一个任务会被线程池执行。
* 5、ASYNC：逻辑实现类似于BACKGROUND，将任务加入到后台的一个队列，最终由Eventbus 中的一个线程池去调用，这里的线程池与 BACKGROUND 逻辑中的线程池用的是同一个，即使用Executors的newCachedThreadPool()方法创建的线程池，它是一个有则用、无则创建、无数量上限的线程池。不同于backgroundPoster的保证任一时间只且仅能有一个任务会被线程池执行的特性，这里asyncPoster则是异步运行的，可以同时接收多个任务。

分析完EventBus的post()方法值，我们接着看看它的unregister()。

#### 四、EventBus.getDefault().unregister(this) <a href="#si-eventbusgetdefaultunregisterthis" id="si-eventbusgetdefaultunregisterthis"></a>

它的核心源码如下所示：

```
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            //1
            unsubscribeByEventType(subscriber, eventType);
        }
        // 2
        typesBySubscriber.remove(subscriber);
    }
}
```

首先，在注释1处，unsubscribeByEventType() 方法中对 subscriptionsByEventType 移除了该 subscriber 的所有订阅信息。最后，在注释2处，移除了注册对象和其对应的所有 Event 事件链表。

最后，我们在来分析下EventBus中对粘性事件的处理。

#### 五、EventBus.getDefault.postSticky(new CollectEvent()) <a href="#wu-eventbusgetdefaultpoststickynewcollectevent" id="wu-eventbusgetdefaultpoststickynewcollectevent"></a>

如果想要发射 sticky 事件需要通过 EventBus的postSticky() 方法，内部源码如下所示：

```
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        // 1
        stickyEvents.put(event.getClass(), event);
    }
    // 2
    post(event);
}
```

在注释1处，先将该事件放入 stickyEvents 中，接着在注释2处使用post()发送事件。前面我们在分析register()方法的最后部分时，其中有关粘性事件的源码如下：

```
if (subscriberMethod.sticky) {
    Object stickyEvent = stickyEvents.get(eventType);
    if (stickyEvent != null) {
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

可以看到，在这里会判断当前事件是否是 sticky 事件，如果 是，则从 stickyEvents 中拿出该事件并执行 postToSubscription() 方法。
