# Android Binder之进程间通信库

> 为了方便对Binder驱动的操作及上层调用，Binder通信的细节及与驱动交互的过程都被封装到了libbinder库中。libbinder相关的源码位于frameworks/native/libs/binder下，本篇分析libbinder库中的基础类。

## RefBase

RefBase在system/core/include/utils/RefBase.h中定义，它是一个公共父类，声明了引用计数操作相关的接口，继承RefBase的子类能通过强指针与弱指针维护它们的生命周期。

## ProcessState

每个使用Binder进程间通信的进程内部都有一个ProcessState对象，它负责与binder驱动建立联系。ProcessState对象通过静态成员函数self()来获取，其成员mDriverFD保存了打开设备文件/dev/binder的句柄值，mVMStart保存映射的内核缓冲区对应用户空间的起始地址。其中mHandleToObject是一个Vector矢量数组，保存着驱动中Binder引用的句柄值及其对应的BpBinder对象(Binder引用即Binder驱动中的binder\_ref，BpBinder即binder的代理类)

```
>>> frameworks/native/include/binder/ProcessState.h

class ProcessState : public virtual RefBase
{
public:
    static  sp<ProcessState>    self();
            ......
            sp<IBinder>         getContextObject(const sp<IBinder>& caller);
            ......
            sp<IBinder>         getContextObject(const String16& name,
                                                 const sp<IBinder>& caller);
            void                startThreadPool();
            ......
private:
            ......
            struct handle_entry {
                IBinder* binder;
                RefBase::weakref_type* refs;
            };

            handle_entry*       lookupHandleLocked(int32_t handle);

            int                 mDriverFD;
            void*               mVMStart;
            ......
            size_t              mMaxThreads;
            ......
            Vector<handle_entry>mHandleToObject;
            ......
};
```

## IPCThreadState

每个使用Binder进程间通信的进程都有一个binder线程池，每个线程都对应一个IPCThreadState对象，它负责与驱动的具体交互，收发进程间通信数据等。IPCThreadState对象通过静态成员函数self()获取，mProcess指向ProcessState对象(每个进程中只存在一个ProcessState对象)。transact()函数负责向驱动发送进程间通信数据，talkWithDriver()负责与驱动的具体交互。

```
>>> frameworks/native/include/binder/IPCThreadState.h

class IPCThreadState
{
public:
    static  IPCThreadState*     self();
            ......
            status_t            transact(int32_t handle,
                                         uint32_t code, const Parcel& data,
                                         Parcel* reply, uint32_t flags);
            ......
private:
            ......
            status_t            talkWithDriver(bool doReceive=true);
            ......
            const   sp<ProcessState>    mProcess;
            const   pid_t               mMyThreadId;
            Parcel              mIn;
            Parcel              mOut;
            const   sp<ProcessState>    mProcess;
            const   pid_t               mMyThreadId;
            ......
};
```

## IBinder

IBinder是一个基础类，BpBinder与BBinder都继承于它，其中成员函数remoteBinder()用于获取远程Binder BpBinder，localBinder()用于获取本地Binder BBinder。

```
>>> frameworks/native/include/binder/IBinder.h

class IBinder : public virtual RefBase
{
public:
    virtual sp<IInterface>  queryLocalInterface(const String16& descriptor);
    ......
    ......
    virtual status_t        transact(   uint32_t code,
                                        const Parcel& data,
                                        Parcel* reply,
                                        uint32_t flags = 0) = 0;
    ......
    virtual BBinder*        localBinder();
    virtual BpBinder*       remoteBinder();
    ......
};
```

## BBinder

BBinder即Binder本地对象，继承于IBinder，它有一个非常重要的成员函数onTransact()，Server收到进程间通信请求后，会调用onTransact()进行分发处理。

```
>>> frameworks/native/include/binder/Binder.h

class BBinder : public IBinder
{
public:
    ......
    virtual BBinder*    localBinder();
protected:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
    ......
};
```

## BpBinder

BpBinder即Binder远程对象，继承于IBinder，它也有一个非常重要的成员函数transact()，BpBinder的transact()会调用IPCThreadState的transact()进行处理。BpBinder中有一成员mHandle，用于保存驱动中Binder引用的句柄，通过这个句柄就可以与驱动中的binder引用关联，其中0代表ServiceManager的binder引用的句柄。当BpBinder通过transact()向驱动发送进程间通信请求时，会将句柄值mHandle发送给驱动，这样驱动就能找到其对应的Binder引用对象，进而找到Binder实体对象，最后就可以找到通信的Service组件了。

```
>>> frameworks/native/include/binder/BpBinder.h

class BpBinder : public IBinder
{
public:
                        BpBinder(int32_t handle);
    inline  int32_t     handle() const { return mHandle; }
    ......
    virtual status_t    transact(   uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
    ......
private:
    const   int32_t             mHandle;
    .....
};
```

## BpRefBase

BpRefBase中有一重要的成员mRemote，它指向BpBinder对象，并可通过成员函数remote()获取。

```
>>> frameworks/native/include/binder/Binder.h

class BpRefBase : public virtual RefBase
{
protected:
                            BpRefBase(const sp<IBinder>& o);
    ......
    inline  IBinder*        remote()                { return mRemote; }
    inline  IBinder*        remote() const          { return mRemote; }

private:
    ......
    IBinder* const          mRemote;
    ......
};
```

## IInterface

IInterface在frameworks/native/include/binder/IInterface.h中定义，它也是一个公共父类。

## BnInterface

BnInterface是一个模板类，同时继承了BBinder和INTERFACE，本地服务通过继承BpInterface来实现。

```
>>> frameworks/native/include/binder/IInterface.h

template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    virtual IBinder*            onAsBinder();
};
```

## BpInterface

BpInterface也是一个模板类，同时继承了BpRefBase和INTERFACE，服务代理通过继承BpInterface来实现。

```
>>> frameworks/native/include/binder/IInterface.h

template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
                                BpInterface(const sp<IBinder>& remote);

protected:
    virtual IBinder*            onAsBinder();
};
```
