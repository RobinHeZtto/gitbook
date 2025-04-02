# Android Binder之Service启动

> Service组件在Server进程启动时会将Service注册到Service Manager中，然后启动Binder线程池等待和处理Client的请求，本篇以MediaPlayerService为例分析Service的启动过程。 相关代码在以下文件中： frameworks/av/media/mediaserver/main\_mediaserver.cpp frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp frameworks/native/libs/binder/IServiceManager.cpp frameworks/native/libs/binder/BpBinder.cpp frameworks/native/libs/binder/Parcel.cpp frameworks/native/libs/binder/IPCThreadState.cpp kernel/drivers/staging/android/binder.c

下图是MediaPlayerService服务注册过程中，MediaPlayerService，Binder Driver，Service Manager之间的一次完整的通信交互过程。

&#x20;!\[]\(https://github.com/RobinHeZtto/Resource/blob/master/blog/image/android/android-binder-addservice.jpg?raw=true)

首先从MediaPlayerService执行入口开始分析：

```
>>> frameworks/av/media/mediaserver/main_mediaserver.cpp

int main(int argc __unused, char **argv __unused)
{
    ......
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());
    ......
    MediaPlayerService::instantiate();
    ......
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

MediaPlayerService进程启动后首先获取其ProcessState对象，然后通过defaultServiceManager()获取到ServiceManager的代理对象，并在创建并初始化MediaPlayerService服务后，通过ServiceManager代理BpServiceManager将MediaPlayerService服务注册到ServiceManager，随后启动线程池并把主线程加入到Binder线程池中等待Client的请求。下面先看MediaPlayerService::instantiate()的过程。

```
>>> frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp

void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```

通过defaultServiceManager()获取的是ServiceManager的代理对象BpServiceManager，MediaPlayerService::instantiate()中通过defaultServiceManager()->addService()将新创建的MediaPlayerService的服务注册到ServiceManager中。defaultServiceManager()->addService()的实现如下：

```
>>> frameworks/native/libs/binder/IServiceManager.cpp

class BpServiceManager : public BpInterface<IServiceManager>
{
public:
    ......
    virtual status_t addService(const String16& name, const sp<IBinder>& service,
            bool allowIsolated)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        data.writeStrongBinder(service);
        data.writeInt32(allowIsolated ? 1 : 0);
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
    ......
};
```

addService()中将进程间通信数据封装成Parcel对象，首先通过writeInterfaceToken写入RPC头信息(RPC包括二部分，第一部分是为Int32类型，描述StrictModePolicy。第二部分为String16类型，描述请求的服务接口即"android.os.IServiceManager")，writeString16(name)写入的是服务的名称，name为"media.player"，writeStrongBinder(service)是将MediaPlayerService封装到flat\_binder\_object结构体中，writeStrongBinder()的实现如下：

```
>>> bionic/libc/kernel/uapi/linux/binder.h

struct flat_binder_object {
  __u32 type;
  __u32 flags;
  union {
    binder_uintptr_t binder;
    __u32 handle;
  };
  binder_uintptr_t cookie;
};

>>> frameworks/native/libs/binder/Parcel.cpp

status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}

status_t flatten_binder(const sp<ProcessState>& /*proc*/, const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        IBinder *local = binder->localBinder();
        if (!local) {
          ......
        } else {
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
          ......
    }

    return finish_flatten_binder(binder, obj, out);
}
```

flatten\_binder中binder->localBinder()为MediaPlayerService的BBinder对象，flat\_binder\_object结构被设置为BINDER\_TYPE\_BINDER类型，obj.binder保存MediaPlayerService的BBinder弱引用，obj.cookie保存MediaPlayerService的BBinder。最后通过finish\_flatten\_binder将flat\_binder\_object写入到Parcel对象out中。 在将IPC数据封装城Parcel对象后，将调用remote()->transact()将数据发送给驱动，根据**Android Binder之Service Manager代理对象**中的分析可知道remote()即BpServiceManager对应的BpBinder对象，下面看BpBinder::transact()的实现。

```
>>> frameworks/native/libs/binder/BpBinder.cpp

status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```

BpBinder::transact()中参数code为ADD\_SERVICE\_TRANSACTION，data为封装成Parcel对象的IPC数据，reply为通信返回，flags表示是否为同步通信等的标志。mAliveBinder代理对象所对应的Binder本地对象是否还活着即ServiceManager是否还活着，mHandle是BpBinder中的句柄，由于当前BpBinder对应于BpServiceManager，所以mHandle值为0。

```
>>> frameworks/native/libs/binder/IPCThreadState.cpp

status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;
    ......
    if (err == NO_ERROR) {
        ......
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    ......
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }

    return err;
}
```

IPCThreadState::transact()中的参数，handle为0，code为ADD\_SERVICE\_TRANSACTION，data为封装MediaPlayerService信息的Parcel对象，reply为接收反馈数据的Parcel对象，flags默认为0。首先检查Parcel对象中的数据的正确性，并设置参数TF\_ACCEPT\_FDS位，代表是否允许server的返回结果中携带文件描述符。然后通过writeTransactionData()将IPC数据进一步封装成binder\_transaction\_data结构并写入mOut缓存中，TF\_ONE\_WAY判断是否是同步的进程间通信请求的标志，最后通过waitForResponse()将数据发送给驱动并等待驱动返回。下面是writeTransactionData()的实现。

```
>>> frameworks/native/libs/binder/IPCThreadState.cpp

status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
      ......
    } else {
      ......
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

writeTransactionData中将传入的参数打包成binder\_transaction\_data对象，tr.target.handle = 0，tr.code = ADD\_SERVICE\_TRANSACTION，tr.flags = TF\_ACCEPT\_FDS。然后将协议命令BC\_TRANSACTION与binder\_transaction\_data结构先后写入到mOut中。

```
>>> frameworks/native/libs/binder/IPCThreadState.cpp

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        ......
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();
        ......
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            ......
            break;

        case BR_DEAD_REPLY:
            ......
            goto finish;

        case BR_FAILED_REPLY:
            ......
            goto finish;

        case BR_ACQUIRE_RESULT:
            ......
            goto finish;

        case BR_REPLY:
            ......
            goto finish;

        default:
            ......
            break;
        }
    }

finish:
    ......
    return err;
}
```

waitForResponse()中循环地调用talkWithDriver与驱动交互，将封装的进程间通信数据发送给驱动并等待驱动返回，下面是talkWithDriver()的实现。

```
>>> frameworks/native/libs/binder/IPCThreadState.cpp

status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ......
    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    ......
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
    ......
#if defined(__ANDROID__)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        ......
    } while (err == -EINTR);

    ......

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        ......
        return NO_ERROR;
    }

    return err;
}
```

talkWithDriver()中使用ioctl BINDER\_WRITE\_READ命令与驱动进行通讯，数据被再次封装成binder\_write\_read结构(IPC数据先是封装成Parcel对象，然后封装成cmd + binder\_transaction\_data对象，最后放到binder\_write\_read中传递给驱动)，其中bwr.write\_buffer，bwr.read\_consumed分别与mOut，mIn对应。最后通过ioctl与发送binder\_write\_read数据给驱动，驱动根据bwr.write\_size > 0调用binder\_thread\_write()处理BC\_TRANSACTION数据，同时根据bwr.read\_size > 0调用binder\_thread\_read()处理返回数据。下面看在binder\_ioctl()中的具体实现。

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
    case BINDER_WRITE_READ:
        ret = binder_ioctl_write_read(filp, cmd, arg, thread);
        if (ret)
            goto err;
        break;
        ......
    }
     ......
    return ret;
}
```

binder\_ioctl()中根据io命令BINDER\_WRITE\_READ，直接进入到binder\_ioctl\_write\_read()进行处理，下面看binder\_ioctl\_write\_read的实现。

```
>>> kernel/drivers/staging/android/binder.c

static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    ......
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }

    if (bwr.write_size > 0) {
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size > 0) {
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
                     bwr.read_size,
                     &bwr.read_consumed,
                     filp->f_flags & O_NONBLOCK);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
        if (ret < 0) {
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }

    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

binder\_ioctl\_write\_read中将用户空间的binder\_write\_read结构的数据拷贝到内核空间bwr中，并根据bwr.write\_size及bwr.read\_size是否大于０，分别调用binder\_thread\_write与binder\_thread\_read进行处理。先看binder\_thread\_write()中的处理。

```
>>> kernel/drivers/staging/android/binder.c

static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        ......
        switch (cmd) {
        ......
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
        ......
        }
        *consumed = ptr - buffer;
    }
    return 0;
}
```

binder\_thread\_write中参数binder\_buffer指向binder\_write\_read中的write\_buffer，从上文已知write\_buffer中存放的是BC\_TRANSACTION + binder\_transaction\_data数据，首先获取到协议命令BC\_TRANSACTION，然后将BC\_TRANSACTION后的数据拷贝到内核空间的binder\_transaction\_data结构中，并调用binder\_transaction()进行处理。

```
>>> kernel/drivers/staging/android/binder.c

static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    binder_size_t *offp, *off_end;
    binder_size_t off_min;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;

    ......
    // reply表示cmd是否是BC_REPLY，根据上文这里获取的cmd是BC_TRANSACTION
    if (reply) {
    ......
    } else {
        // 根据target的句柄找到目标binder_ref进而找到binder_node及binder_proc，由于是与ServiceManager通信，这里的handle值为0
        if (tr->target.handle) {
            struct binder_ref *ref;
            // 根据handle句柄值在proc->refs_by_desc上查找其对应的binder_ref
            ref = binder_get_ref(proc, tr->target.handle);
            ......
            target_node = ref->node;
        } else {
            // Service Manager的binder实体为binder_context_mgr_node
            target_node = binder_context_mgr_node;
            ......
        }
        // 根据target的binder_node找到处理事务的目标进程
        target_proc = target_node->proc;
        ......
    }
    //如果找到目标线程target_thread处理事务，则target_list，target_wait指向target_thread，否则指向target_proc
    if (target_thread) {
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    // 分配一个待处理的事务binder_transaction对象
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    ......
    // 分配一个待完成的工作tcomplete binder_work对象
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    ......
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;
    // 初始化binder_transaction对象各成员
    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc;
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);
    // 为target_proc分配内核缓冲区，以便接受IPC数据
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    if (t->buffer == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_binder_alloc_buf_failed;
    }
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL);

    offp = (binder_size_t *)(t->buffer->data +
                 ALIGN(tr->data_size, sizeof(void *)));
    // 将IPC数据拷贝到target_proc的内核缓冲区
    if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
               tr->data.ptr.buffer, tr->data_size)) {
        ......           
        goto err_copy_data_failed;
    }
    if (copy_from_user(offp, (const void __user *)(uintptr_t)
               tr->data.ptr.offsets, tr->offsets_size)) {
        ......    
        goto err_copy_data_failed;
    }
    ......    
    off_end = (void *)offp + tr->offsets_size;
    off_min = 0;
    // 循环处理IPC数据中的Binder对象
    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        ......
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        off_min = *offp + sizeof(struct flat_binder_object);
        // 从上文可知类型为BINDER_TYPE_BINDER
        switch (fp->type) {
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct binder_ref *ref;
            // 根据BBinder的引用从当前的binder_proc中的获取对应的binder_node
            struct binder_node *node = binder_get_node(proc, fp->binder);
            // 根据上文可知，这里node == NULL，所以为MediaPlayerService新建binder实体binder_node
            if (node == NULL) {
                node = binder_new_node(proc, fp->binder, fp->cookie);
                ......
                node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
                node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
            }
            ......
            // 在ServiceManager的binder_proc中找对应MediaPlayerService的binder_ref，如果没有找到则为其新建binder_ref加入到其红黑树中(同时为binder_ref分配句柄值)
            ref = binder_get_ref_for_node(target_proc, node);
            if (ref == NULL) {
                return_error = BR_FAILED_REPLY;
                goto err_binder_get_ref_for_node_failed;
            }
            if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
            else
                fp->type = BINDER_TYPE_WEAK_HANDLE;
            fp->binder = 0;
            fp->handle = ref->desc;
            fp->cookie = 0;
            binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
        } break;
        ......
    }
    if (reply) {
    ......
    // 根据TF_ONE_WAY判断是否是同步通信
    } else if (!(t->flags & TF_ONE_WAY)) {
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
        if (target_node->has_async_transaction) {
            target_list = &target_node->async_todo;
            target_wait = NULL;
        } else
            target_node->has_async_transaction = 1;
    }
    // 将t封装成BINDER_WORK_TRANSACTION的工作项，放到ServiceManager的proc或thread的todo列表中
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    // 将tcomplete置为BINDER_WORK_TRANSACTION_COMPLETE，并放在源线程的todo列表中
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒目标进程
    if (target_wait) {
        if (reply || !(t->flags & TF_ONE_WAY)) {
            preempt_disable();
            wake_up_interruptible_sync(target_wait);
            preempt_enable_no_resched();
        }
        else {
            wake_up_interruptible(target_wait);
        }
    }
    return;
}
```

binder\_transaction()的主要工作是解析binder\_transaction，根据通信目标handle值tr->target.handle找到其对应的binder\_ref，通过binder\_ref找到binder\_node，进而找到目标进程binder\_proc。在MediaPlayerService向ServiceManager注册时，handle值为0，可直接找到ServiceManager的binder实体binder\_context\_mgr\_node，进而找到其binder\_proc，同时为MediaPlayerService新建binder\_node实体，然后将其引用binder\_ref挂到ServiceManager的binder\_proc的引用列表上。然后查找ServiceManager的空闲线程，如果找到其空闲的线程，将封装成BINDER\_WORK\_TRANSACTION类型的binder\_transaction事务放到binder\_thread的todo列表上，如果没有找到空闲的线程，则放在binder\_proc的todo列表上。同时也将封装BINDER\_WORK\_TRANSACTION\_COMPLETE类型的binder\_work放到ServiceManager的proc或thread的todo列表中，然后唤醒目标进程执行。 此时，源线程与目标线程都会去执行其todo队列中待处理的事物，下面先看源线程MediaPlayerService中的处理，再看目标进程ServiceManager中的处理。源线程从binder\_transaction()返回到binder\_thread\_write()然后返回到binder\_ioctl\_write\_read()，继续执行binder\_ioctl\_write\_read()中的binder\_thread\_read()。

```
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    ......

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        if (!list_empty(&thread->todo)) {
            w = list_first_entry(&thread->todo, struct binder_work,
                         entry);
        } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
            w = list_first_entry(&proc->todo, struct binder_work,
                         entry);
        } else {
            /* no data added */
            if (ptr - buffer == 4 &&
                !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
                goto retry;
            break;
        }
        ......
        case BINDER_WORK_TRANSACTION_COMPLETE: {
            cmd = BR_TRANSACTION_COMPLETE;
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);

            binder_stat_br(proc, thread, cmd);

            list_del(&w->entry);
            kfree(w);
            binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
        } break;　
        ......
        }

        ......
done:
    *consumed = ptr - buffer;
    ......
}
```

binder\_ioctl\_write\_read()中从thread->todo或proc->todo读取到BINDER\_WORK\_TRANSACTION\_COMPLETE工作项，写入到用户空间的bwr.read\_buffer中并返回到用户空间，接下来便又回到waitForResponse中处理。 接下来看ServiceManager端的处理，在**Android Binder之Service Manager**中已经分析到ServiceManager启动后进入到binder\_thread\_read()，等待其todo队列上的事务。前面已分析到Binder Driver已经将一个BINDER\_WORK\_TRANSACTION的工作项加入到ServiceManager的todo队列中，那么ServiceManager被唤醒处理该事务，下面从ServiceManager的binder\_thread\_read()开始分析其过程。

```
>>> kernel/drivers/staging/android/binder.c

static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    ......
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        // 检查thread->todo上是否有工作项需要处理
        if (!list_empty(&thread->todo)) {
            w = list_first_entry(&thread->todo, struct binder_work,
                         entry);
        // 检查proc->todo上是否有工作项需要处理
        } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
            w = list_first_entry(&proc->todo, struct binder_work,
                         entry);
        } else {
            /* no data added */
            if (ptr - buffer == 4 &&
                !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
                goto retry;
            break;
        }
        ......
        switch (w->type) {
        // 前文已知，BINDER_WORK_TRANSACTION类型的工作项被加入到ServiceManager的todo队列中
        case BINDER_WORK_TRANSACTION: {
            // 根据binder_transaction中的work成员获取binder_transaction
            t = container_of(w, struct binder_transaction, work);
        } break;
        ......
        // target_node指向binder_context_mgr_node
        if (t->buffer->target_node) {
            struct binder_node *target_node = t->buffer->target_node;
            // 构造BR_TRANSACTION类型的binder_transaction_data
            tr.target.ptr = target_node->ptr;
            tr.cookie =  target_node->cookie;
            t->saved_priority = task_nice(current);
            if (t->priority < target_node->min_priority &&
                !(t->flags & TF_ONE_WAY))
                binder_set_nice(t->priority);
            else if (!(t->flags & TF_ONE_WAY) ||
                 t->saved_priority > target_node->min_priority)
                binder_set_nice(target_node->min_priority);
            cmd = BR_TRANSACTION;
        } else {
            tr.target.ptr = 0;
            tr.cookie = 0;
            cmd = BR_REPLY;
        }
        tr.code = t->code;
        tr.flags = t->flags;
        tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);

        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;

            tr.sender_pid = task_tgid_nr_ns(sender,
                            task_active_pid_ns(current));
        } else {
            ......
        }

        tr.data_size = t->buffer->data_size;
        tr.offsets_size = t->buffer->offsets_size;
        tr.data.ptr.buffer = (binder_uintptr_t)(
                    (uintptr_t)t->buffer->data +
                    proc->user_buffer_offset);
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));
        // 将tr拷贝到用户空间缓冲区
        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        if (copy_to_user(ptr, &tr, sizeof(tr)))
            return -EFAULT;
        ptr += sizeof(tr);

        ......

        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
            t->to_parent = thread->transaction_stack;
            t->to_thread = thread;
            thread->transaction_stack = t;
        } else {
            t->buffer->transaction = NULL;
            kfree(t);
            binder_stats_deleted(BINDER_STAT_TRANSACTION);
        }
        break;
    }
    ......
    return 0;
}
```

ServiceManager的主线程在binder\_thread\_read()中被唤醒后，将回到用户空间处理收到的IPC数据，调用binder\_parse()进行处理。

```
>>> frameworks/native/cmds/servicemanager/binder.c

int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
        ......
        switch(cmd) {
        ......
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
                // 消息处理
                res = func(bs, txn, &msg, &reply);
                if (txn->flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
                }
            }
            ptr += sizeof(*txn);
            break;
        }
        ......
    }

    return r;
}
```

binder\_parse()中读取cmd与binder\_transaction\_data数据，并调用svcmgr\_handler()进行处理。

```
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;

    ......

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }

    if ((len != (sizeof(svcmgr_id) / 2)) ||
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s\n", str8(s, len));
        return -1;
    }

    ......

    switch(txn->code) {
    ......
    case SVC_MGR_ADD_SERVICE:
        // 获取注册服务的名称
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        // 获取注册服务的句柄
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        // 注册服务
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;
    ......
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

最终，服务将被注册到svclist链表中。前面说到Service组件在Server进程启动时会将Service注册到Service Manager，然后启动Binder线程池等待并处理Client的请求，下面分析Binder线程池的启动过程。

```
>>> frameworks/av/media/mediaserver/main_mediaserver.cpp

int main(int argc __unused, char **argv __unused)
{
    ......
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());
    ......
    MediaPlayerService::instantiate();
    ......
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

回到MediaPlayerService进程的main()函数，通过ProcessState::self()->startThreadPool()启动线程池，并通过IPCThreadState::self()->joinThreadPool()将当前线程加入到线程池中等待并处理客户端的请求。先看startThreadPool()的实现。

```
>>> frameworks/native/libs/binder/ProcessState.cpp

void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
```

startThreadPool()中通过mThreadPoolStarted判断是否已经启动线程池，调用spawnPooledThread()启动线程池。

```
>>> frameworks/native/libs/binder/ProcessState.cpp

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```

spawnPooledThread()中创建一个PoolThread对象(PoolThread继承与Thread)，并通过其run函数来启动一个新线程。接下来继续看

```
>>> frameworks/native/libs/binder/IPCThreadState.cpp

void IPCThreadState::joinThreadPool(bool isMain)
{
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    ......

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```

最终，MediaPlayerService通过向驱动发送BC\_ENTER\_LOOPER完成Binder线程注册的过程。
