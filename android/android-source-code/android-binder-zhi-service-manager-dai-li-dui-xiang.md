# Android Binder之Service Manager代理对象

> Service组件在启动时需要将自己注册到Server Manager，Client组件在使用Service的服务前也需要从Service Manager获取到Service的代理对象，但由于Service Manager运行在一个独立的进程当中，Client组件或Service组件在请求Service Manager服务前，首先得获取到Service Manager的代理对象，本篇主要分析Service Manager代理对象获取的流程。
>
> 相关的源码在以下文件中： frameworks/native/include/binder/IServiceManager.h frameworks/native/libs/binder/IServiceManager.cpp frameworks/native/libs/binder/Static.cpp frameworks/native/libs/binder/ProcessState.cpp frameworks/native/libs/binder/BpBinder.cpp frameworks/native/libs/binder/IPCThreadState.cpp frameworks/native/include/binder/IInterface.h

Service Manager代理对象类型为BpServiceManager，它继承于BpInterface并实现了IServiceManager接口，Frameworks层libbinder提供defaultServiceManager接口用来获取BpServiceManager对象。下图是defaultServiceManager()的执行流程。  接下来按照上图的流程进行分析，首先看一下IServiceManager的实现。

![defaultServiceManager执行流程](https://github.com/RobinHeZtto/Resource/blob/master/blog/image/android/android-defaultServiceManager.jpg?raw=true)

```
>>> frameworks/native/include/binder/IServiceManager.h

class IServiceManager : public IInterface
{
public:
    DECLARE_META_INTERFACE(ServiceManager);

    virtual sp<IBinder>         getService( const String16& name) const = 0;

    virtual sp<IBinder>         checkService( const String16& name) const = 0;

    virtual status_t            addService( const String16& name,
                                            const sp<IBinder>& service,
                                            bool allowIsolated = false) = 0;

    virtual Vector<String16>    listServices() = 0;

    enum {
        GET_SERVICE_TRANSACTION = IBinder::FIRST_CALL_TRANSACTION,
        CHECK_SERVICE_TRANSACTION,
        ADD_SERVICE_TRANSACTION,
        LIST_SERVICES_TRANSACTION,
    };
};
```

IServiceManager中定义了getService，checkService，addService，listServices四个成员函数，即Service Manager提供给client用来获取/注册服务的的接口，client获取到Service Manager代理对象后，就能通过这些接口获取注册到Service Manager中服务的信息。下面开始分析如何通过defaultServiceManager()获取Service Manager代理对象。

```
>>> defaultServiceManager()在frameworks/native/libs/binder/IServiceManager.cpp

sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;

    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }

    return gDefaultServiceManager;
}
```

gDefaultServiceManager是指向Service Manager代理对象BpServiceManager的强指针，一个进程只存在一个BpServiceManager代理对象，defaultServiceManager()中通过gDefaultServiceManager判断是否已经创建过Service Manager代理对象，如果已经创建过，则直接返回其指针gDefaultServiceManager，否则为获取当前进程的Service Manager代理对象并保存在gDefaultServiceManager中。

```
>>> frameworks/native/libs/binder/Static.cpp

Mutex gDefaultServiceManagerLock;
sp<IServiceManager> gDefaultServiceManager;
```

defaultServiceManager()中的gDefaultServiceManager的获取即： `gDefaultServiceManager = interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL));` 首先看ProcessState::self()的实现。

```
>>> frameworks/native/libs/binder/ProcessState.cpp

sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState;
    return gProcess;
}
```

ProcessState::self()即返回ProcessState的对象，每个进程只存在一个ProcessState对象。

```
>>> frameworks/native/libs/binder/Static.cpp

Mutex gProcessMutex;
sp<ProcessState> gProcess;
```

继续看new ProcessState的过程。

```
>>> frameworks/native/libs/binder/ProcessState.cpp

#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))
#define DEFAULT_MAX_BINDER_THREADS 15

ProcessState::ProcessState()
    : mDriverFD(open_driver())
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
        }
    }
}

static int open_driver()
{
    int fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            ALOGE("Binder driver protocol does not match user space protocol!");
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
    }
    return fd;
}
```

ProcessState()的构造函数中主要执行的是对/dev/binder的操作，打开/dev/binder并将fd保存在mDriverFD中，通过ioctl设置binder通信线程池最大线程数为BINDER\_SET\_MAX\_THREADS 15个，mmap映射BINDER\_VM\_SIZE 1016kb大小缓冲区。open()，mmap()在驱动章节已分析过，下面分析ioctl() BINDER\_SET\_MAX\_THREADS的流程。

```
>>> kernel/drivers/staging/android/binder.c

static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    ......
    binder_lock(__func__);
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    ......
    case BINDER_SET_MAX_THREADS:
        if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
            ret = -EINVAL;
            goto err;
        }
        break;
        .....
    return ret;
}
```

binder\_ioctl BINDER\_SET\_MAX\_THREADS即设置proc->max\_threads。继续看ProcessState::self()->getContextObject(NULL)的实现。

```
>>> frameworks/native/libs/binder/ProcessState.cpp

sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}
```

getContextObject返回的是sp对象，即返回的是Service Manager的BpBinder对象，直接通过getStrongProxyForHandle(0)实现，Service Manager作为一个特殊的Service组件，它对应的句柄值是０。

```
>>> frameworks/native/libs/binder/ProcessState.cpp

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }

            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

每个ProcessState中都维护一个了handle\_entry类型的BpBinder代理对象列表，以句柄值为索引

```
class ProcessState : public virtual RefBase
{


private:
            ......

            struct handle_entry {
                IBinder* binder;
                RefBase::weakref_type* refs;
            };

            handle_entry*       lookupHandleLocked(int32_t handle);

            ......
};
```

getStrongProxyForHandle()通过lookupHandleLocked()根据句柄值在mHandleToObject中查找对应的handle\_entry，如果不存在，则新建handle\_entry并保存在mHandleToObject中。

```
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}
```

找到handle\_entry后判断handle\_entry->binder是否存在，如果不存在，则根据句柄值handle创建新的BpBinder对象。

```
>>> frameworks/native/libs/binder/BpBinder.cpp

BpBinder::BpBinder(int32_t handle)
    : mHandle(handle)
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);
    IPCThreadState::self()->incWeakHandle(handle);
}
```

将句柄handle保存到私有成员mHandle中，并通过IPCThreadState增加Service Manager实体的引用计数。

```
>>> frameworks/native/libs/binder/IPCThreadState.cpp

IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;
    }

    if (gShutdown) {
        ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash.\n");
        return NULL;
    }

    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s\n",
                    strerror(key_create_value));
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```

IPCThreadState::self中获取IPCThreadState对象。若该对象已经存在，则直接返回，否则新建IPCThreadState对象。 至此，`ProcessState::self()->getContextObject(NULL))`执行完成，它返回的是Service Manager的Bpbinder对象，即`gDefaultServiceManager = interface_cast<IServiceManager>(Bpbinder(0))`，继续看interface\_cast()的实现。

```
>>> frameworks/native/include/binder/IInterface.h

template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```

即IServiceManager::asInterface(Bpbinder(0))，但asInterface并未直接在IServiceManager中定义，而是通过宏 **DECLARE\_META\_INTERFACE(ServiceManager)**, **IMPLEMENT\_META\_INTERFACE(ServiceManager, "android.os.IServiceManager")** 来实现的。

```
>>> frameworks/native/include/binder/IInterface.h

#define DECLARE_META_INTERFACE(INTERFACE)                               \
    static const android::String16 descriptor;                          \
    static android::sp<I##INTERFACE> asInterface(                       \
            const android::sp<android::IBinder>& obj);                  \
    virtual const android::String16& getInterfaceDescriptor() const;    \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();                                            \

#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const android::String16 I##INTERFACE::descriptor(NAME);             \
    const android::String16&                                            \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<I##INTERFACE> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }                                   \
```

将INTERFACE以ServiceManager替换展开如下：

```
#define DECLARE_META_INTERFACE(ServiceManager)                               \
    static const android::String16 descriptor;                          \
    static android::sp<IServiceManager> asInterface(                       \
            const android::sp<android::IBinder>& obj);                  \
    virtual const android::String16& getInterfaceDescriptor() const;    \
    IServiceManager;                                                     \
    virtual ~IServiceManager();                                            \


#define IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager")                       \
    const android::String16 IServiceManager::descriptor("android.os.IServiceManager");             \
    const android::String16&                                            \
            IServiceManager::getInterfaceDescriptor() const {              \
        return IServiceManager::descriptor;                                \
    }                                                                   \
    android::sp<IServiceManager> IServiceManager::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<IServiceManager> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<IServiceManager*>(                          \
                obj->queryLocalInterface(                               \
                        IServiceManager::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new BpServiceManager(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    IServiceManager::IServiceManager() { }                                    \
    IServiceManager::~IServiceManager() { }                                   \
```

IServiceManager::asInterface()的实现如下，asInterface()即根据Bpbinder对象获取IServiceManager。

```
android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)              
{                                                                
    android::sp<IServiceManager> intr;                                
    if (obj != NULL) {                                              
        intr = static_cast<IServiceManager*>(                          
            obj->queryLocalInterface(                               
                    IServiceManager::descriptor).get());               
        if (intr == NULL) {                                         
            intr = new BpServiceManager(obj);                          
        }                                                           
    }                                                               
    return intr;                                                    
}
```

参数obj即BpBinder(0)，obj->queryLocalInterface("android.os.IServiceManager")查找名称为"android.os.IServiceManager"的本地接口，queryLocalInterface()的实现在BpBinder的父类IBinder中，现在IServiceManager接口还没创建，intr=NULL，根据BpBinder创建一个BpServiceManager，即gDefaultServiceManager = new BpServiceManager(new BpBinder(0))。

```
>>> frameworks/native/libs/binder/IServiceManager.cpp

 BpServiceManager(const sp<IBinder>& impl)
    : BpInterface<IServiceManager>(impl)
{
}

>>> frameworks/native/include/binder/IInterface.h

template<typename INTERFACE>
inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote)
    : BpRefBase(remote)
{
}

>>> frameworks/native/include/binder/Binder.h

class BpRefBase : public virtual RefBase
{
    ......
private:
    ......
    IBinder* const          mRemote;
    ......
};
```

由上可知BpServiceManager继承于BpInterface，而BpInterface又继承于BpRefBase，最后BpBinder(0)即mRemote对象。
