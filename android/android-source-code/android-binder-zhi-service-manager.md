# Android Binder之Service Manager

> ServiceManager是Binder进程间通信的核心组件，负责管理Service组件并向Client提供Service代理对象。ServiceManager运行在独立的进程中，Service或Client组件也需要与ServiceManager进行进程间通信，所以ServiceManager除了是Binder通信的上下文管理者外，也是一个特殊的Service组件。
>
> ServiceManager源码位于以下文件中：&#x20;
>
> frameworks/native/cmds/servicemanager/service\_manager.c frameworks/native/cmds/servicemanager/binder.c frameworks/native/cmds/servicemanager/binder.h kernel/drivers/staging/android/binder.c

## 1. 启动脚本

ServiceManager是由Init进程通过解析Initrc启动的，它的启动脚本如下：

```
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm

on boot
    .......
    class_start core
```

servicemanager服务定义在`class core`中，Init进程解析执行Initrc的`on boot`阶段时会通过`class_start core`启动包括servicemanager在内的系统核心服务。其中`user system`与`group system`指定servicemanager以system的身份运行。critical表明servicemanager是系统的关键服务，关键服务如果退出4次以上，系统将重启至recovery模式。最后的`onrestart`表示servicemanager退出后需要重新启动healthd，zygote，media，surfaceflinger，drm服务。

## 2. 启动流程

servicemanager的源码位于frameworks/native/cmds/servicemanager下,启动入口是service\_manager.c的main()函数,下图是servicemanager的大致启动流程.

&#x20;!\[service\_manager]\(https://github.com/XRobinHe/Resource/blob/master/blog/image/android/binder\_service\_manager.png?raw=true) 下面从main()开始分析servicemanager的启动过程.

![](<../../.gitbook/assets/image (383).png>)

```
>>> frameworks/native/cmds/servicemanager/service_manager.c

int main()
{
    struct binder_state *bs;

    bs = binder_open(128*1024);
    if (!bs) {
        ALOGE("failed to open binder driver\n");
        return -1;
    }

    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    ......

    binder_loop(bs, svcmgr_handler);

    return 0;
}
```

整个启动过程分成三步，首先binder\_open打开/dev/binder并mmap映射到本进程地址空间，然后调用binder\_become\_context\_manager注册成为整个binder通信系统的管理者，最后在binder\_loop中循环等待并处理client的请求。下面逐步进行分析。

### binder\_open

首先看一下binder\_state结构体，它在binder.c中定义。

```
>>> frameworks/native/cmds/servicemanager/binder.c

struct binder_state
{
    int fd;  // 打开/dev/binder的文件描述符
    void *mapped; // mmap的起始地址
    size_t mapsize; // mmap的大小
};
```

binder\_state主要用来保存binder open及ioctl操作的信息。下面看binder\_open的实现，binder\_open也在binder.c中实现。

```
>>> frameworks/native/cmds/servicemanager/binder.c

struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

    bs->fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }

    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```

binder\_open主要完成的是打开/dev/binder并映射到本进程的地址空间，对应的信息保存在binder\_state中，（open mmap分别会调到driver层的binder\_open与binder\_mmap，binder\_open与binder\_mmap的具体实现可参考上篇）。其中mmap，第一个参数是映射内存的起始地址，NULL表示系统指定地址，mapsize大小是128\*1024B，即为servicemanager分配了128K内核缓冲区。PROT\_READ表示映射区域是可读的，MAP\_PRIVATE表示建立一个写入时拷贝的私有映射，即当进程中对该内存区域进行写入时，是写入到映射的拷贝中。

### binder\_become\_context\_manager

binder\_become\_context\_manager同样在binder.c中实现，通过调用该函数使servicemanager注册成为Binder的上下文管理者。

```
>>> frameworks/native/cmds/servicemanager/binder.c

int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}

>>> drivers/staging/android/uapi/binder.h
#define BINDER_SET_CONTEXT_MGR      _IOW('b', 7, __s32)
```

binder\_become\_context\_manager通过ioctl设置BINDER\_SET\_CONTEXT\_MGR，servicemanager作为一个特殊的service，BINDER\_SET\_CONTEXT\_MGR的参数设置为０。

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
    // 从proc获取对应当前线程的binder_thread，没有则创建并关联到proc
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    ......
    case BINDER_SET_CONTEXT_MGR:
        // 已经存在binder上下文管理者的binder实体，说明已经注册，出错返回
        if (binder_context_mgr_node != NULL) {
            pr_err("BINDER_SET_CONTEXT_MGR already set\n");
            ret = -EBUSY;
            goto err;
        }
        ret = security_binder_set_context_mgr(proc->tsk);
        if (ret < 0)
            goto err;
        if (uid_valid(binder_context_mgr_uid)) {
            if (!uid_eq(binder_context_mgr_uid, current->cred->euid)) {
                pr_err("BINDER_SET_CONTEXT_MGR bad uid %d != %d\n",
                       from_kuid(&init_user_ns, current->cred->euid),
                       from_kuid(&init_user_ns, binder_context_mgr_uid));
                ret = -EPERM;
                goto err;
            }
        } else
            // binder_context_mgr_uid设置为当前进程的有效euid
            binder_context_mgr_uid = current->cred->euid;
            // 创建binder上下文管理者的binder实体，并设置binder_context_mgr_node
            binder_context_mgr_node = binder_new_node(proc, 0, 0);
        if (binder_context_mgr_node == NULL) {
            ret = -ENOMEM;
            goto err;
        }
        // 增加引用计数，防止驱动将其释放
        binder_context_mgr_node->local_weak_refs++;
        binder_context_mgr_node->local_strong_refs++;
        binder_context_mgr_node->has_strong_ref = 1;
        binder_context_mgr_node->has_weak_ref = 1;
        break;
    ......
    }
    ret = 0;
err:
    if (thread)
        // 清除thread->looper上的BINDER_LOOPER_STATE_NEED_RETURN状态
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}
```

1. 首先通过binder\_get\_thread获取当前线程对应的binder\_thread，即在红黑树proc->threads中以pid为关键字遍历，如果没有则创建一个。
2. 检查binder\_context\_mgr\_node是否为NULL，防止重复注册。
3.  然后设置binder\_context\_mgr\_uid为当前uid，并通过binder\_new\_node新建binder实体并赋值给binder\_context\_mgr\_node，新建Binder实体后并增加其引用计数防止驱动将其释放。最后清除thread BINDER\_LOOPER\_STATE\_NEED\_RETURN状态返回。下面是binder\_new\_node的实现：

    ```
    static struct binder_node *binder_new_node(struct binder_proc *proc,
                        binder_uintptr_t ptr,
                        binder_uintptr_t cookie)
    {
     struct rb_node **p = &proc->nodes.rb_node;
     struct rb_node *parent = NULL;
     struct binder_node *node;

     while (*p) {
         parent = *p;
         node = rb_entry(parent, struct binder_node, rb_node);

         if (ptr < node->ptr)
             p = &(*p)->rb_left;
         else if (ptr > node->ptr)
             p = &(*p)->rb_right;
         else
             return NULL;
     }

     node = kzalloc_preempt_disabled(sizeof(*node));
     if (node == NULL)
         return NULL;
     binder_stats_created(BINDER_STAT_NODE);
     rb_link_node(&node->rb_node, parent, p);
     rb_insert_color(&node->rb_node, &proc->nodes);
     node->debug_id = ++binder_last_id;
     node->proc = proc;
     node->ptr = ptr;
     node->cookie = cookie;
     node->work.type = BINDER_WORK_NODE;
     INIT_LIST_HEAD(&node->work.entry);
     INIT_LIST_HEAD(&node->async_todo);
     return node;
    }
    ```

    binder\_new\_node()有三个参数，proc即当前进程在kernel中的上下文描述binder\_proc，ptr与cookie用来描述binder本地对象，由于是service manager，对应的binder本地对象地址值为0。binder\_new\_node根据ptr在proc的node红黑书中索引，如果找到已经创建的binder\_node实体返回NULL，没有找到则创建binder\_node实体对象。

### biner\_loop

打开Binder设备，映射到进程地址空间并成为进程间通信上下文管理者后，servicemanager将进入到无限循环中等待client的请求。

```
>>> kernel/drivers/staging/android/binder.c

void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    // 通过ioctl BC_ENTER_LOOPER将自己注册为binder线程，以便binder驱动分发请求
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }

        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```

binder\_loop中先通过ioctl BC\_ENTER\_LOOPER注册binder线程，然后在无限循环中ioctl BINDER\_WRITE\_READ等待请求数据，并通过binder\_parse处理client请求。binder\_loop中的处理在后面分析业务时具体分析，下面具体看binder\_loop中二次ioctl的实现，分别是写BC\_ENTER\_LOOPER，等待读取客户端数据。

```
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
    case BINDER_SET_MAX_THREADS:
    ......
        break;
    case BINDER_SET_CONTEXT_MGR:
    ......
        break;
    case BINDER_THREAD_EXIT:
    ......
        break;
    case BINDER_VERSION:
    ......
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
err_unlocked:
    return ret;
}
```

binder\_ioctl根据cmd BINDER\_WRITE\_READ进入到binder\_ioctl\_write\_read执行。

```
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

    if (bwr.write_size > 0) {
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user_preempt_disabled(ubuf, &bwr, sizeof(bwr)))
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
            if (copy_to_user_preempt_disabled(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
  ......
out:
    return ret;
}
```

根据bwr.write\_size与bwr.read\_size判断，当bwr.write\_size大于０时，执行binder\_thread\_write发送数据，当binder\_thread\_read大与０时，执行binder\_thread\_read等待读取数据。先看binder\_thread\_write的实现。

```
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    struct binder_context *context = proc->context;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user_preempt_disabled(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    ......
        switch (cmd) {
        case BC_INCREFS:
        case BC_ACQUIRE:
        case BC_RELEASE:
        case BC_DECREFS:
      ......
        case BC_INCREFS_DONE:
        case BC_ACQUIRE_DONE:
      ......
        case BC_ATTEMPT_ACQUIRE:
      ......
        case BC_ACQUIRE_RESULT:
      ......
        case BC_FREE_BUFFER:      
      ......
        case BC_TRANSACTION_SG:
        case BC_REPLY_SG:
      ......
        case BC_TRANSACTION:
        case BC_REPLY:
      ......
        case BC_REGISTER_LOOPER:
      ......
        case BC_ENTER_LOOPER:
            if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("%d:%d ERROR: BC_ENTER_LOOPER called after BC_REGISTER_LOOPER\n",
                    proc->pid, thread->pid);
            }
            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
        case BC_EXIT_LOOPER:
      ......
        case BC_REQUEST_DEATH_NOTIFICATION:
        case BC_CLEAR_DEATH_NOTIFICATION:
      ......
      break;
        case BC_DEAD_BINDER_DONE:
      ......
        default:
            return -EINVAL;
        }
        *consumed = ptr - buffer;
    }
    return 0;
}
```

binder\_thread\_write中对BC\_ENTER\_LOOPER的处理即把thread->looper的状态置为BINDER\_LOOPER\_STATE\_ENTERED，标志ServiceManager主线程已经准备好可以处理Client的请求了。

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
    int wait_for_proc_work;

    if (*consumed == 0) {
        if (put_user_preempt_disabled(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
  // 线程的事物堆栈为NULL，且todo队列为空时，表示当前线程空闲
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);
  ......
  // 设置线程的状态为BINDER_LOOPER_STATE_WAITING
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
        proc->ready_threads++;

    binder_unlock(__func__);

    if (wait_for_proc_work) {
    ......
        binder_set_nice(proc->default_priority);
        if (non_block) {
            // 非阻塞式的读取，则通过binder_has_proc_work()读取proc的事务；
            // 若没有，则直接返回
            if (!binder_has_proc_work(proc, thread))
                ret = -EAGAIN;
        } else
            // 阻塞式的读取，则阻塞等待事务的发生。
            ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }

    binder_lock(__func__);

    if (wait_for_proc_work)
        proc->ready_threads--;
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        return ret;

    // 处理工作项中的数据    
    while (1) {
    ......
    }

done:

    *consumed = ptr - buffer;
  ......
    return 0;
}
```
