# Android Binder之Binder Driver

> Binder驱动源码位于kernel/drivers/staging/android/目录下，包括binder.c与uapi/binder.h，主要负责通信数据传递等底层实现。

## 1. 数据结构

在分析驱动代码之前，首先需要熟悉Binder驱动相关的数据结构。

### binder\_node

binder\_node在binder.c中定义，它是binder实体对象，代表着Service组件或ServiceManager在内核中的描述，可以通过binder\_node找到用户空间的Service对象。

```
struct binder_node {
    int debug_id;
    struct binder_work work;
    union {
        struct rb_node rb_node;
        struct hlist_node dead_node;
    };
    struct binder_proc *proc;
    struct hlist_head refs;　
    int internal_strong_refs;
    int local_weak_refs;
    int local_strong_refs;
    binder_uintptr_t ptr;
    binder_uintptr_t cookie;
    unsigned has_strong_ref:1;
    unsigned pending_strong_ref:1;
    unsigned has_weak_ref:1;
    unsigned pending_weak_ref:1;
    unsigned has_async_transaction:1;
    unsigned accept_fds:1;
    unsigned min_priority:8;
    struct list_head async_todo;
};
```

binder\_node是描述Binder实体的数据结构，Binder驱动会为每个Service创建一个Binder实体。&#x20;

```
    union {
        struct rb_node rb_node;
        struct hlist_node dead_node;
    };
```

binder\_node未销毁时，rb\_node链接到proc->nodes红黑树上. binder\_node销毁后，dead\_node加入到全局的死亡hash链表中.&#x20;

```
struct binder_proc *proc;
```

Binder实体(服务)所在的宿主进程（进程上下文信息）.

```
   struct hlist_head refs;　
```

Binder实体引用的链表，所有该binder实体的引用都保存在该hash list中.&#x20;

```
    binder_uintptr_t ptr;
```

描述用户空间的Service組件，对应Binder实体对应的Service在用户空间的本地Binder(BBinder)的引用&#x20;

```
    binder_uintptr_t cookie;
```

描述用户空间的Service組件，Binder实体对应的Service在用户空间的本地Binder(BBinder)地址

### binder\_ref

binder\_ref在binder.c中定义，它是Client组件在内核中的描述，通过binder引用可以找到其在内核中的binder实体，进而找到用户空间的Service对象。

```
struct binder_ref {
    int debug_id;
    struct rb_node rb_node_desc;　
    struct rb_node rb_node_node;　
    struct hlist_node node_entry;　
    struct binder_proc *proc;　
    struct binder_node *node;
    uint32_t desc;
    int strong;
    int weak;
    struct binder_ref_death *death;
};
```

binder\_ref是描述Binder引用的数据结构，Binder驱动会为每个Client创建一个Binder引用。&#x20;

以句柄值作为关键字关联到binder\_proc->refs\_by\_desc红黑树 :

```
struct rb_node rb_node_desc;　
```

以Binder实体对象地址作为关键字关联到binder\_proc->refs\_by\_node红黑树 :

```
struct rb_node rb_node_node;
```

binder\_node的引用节点，关联到binder\_node->refs :

```
struct hlist_node node_entry;
```

引用所在的宿主进程 :

```
struct binder_proc *proc;　
```

引用所对应的binder\_node实体

```
struct binder_node *node;
```

Binder引用的句柄值，Binder驱动为binder引用分配的一个唯一的int型整数（进程范围内唯一），通过该值可以在binder\_proc->refs\_by\_desc中找到Binder引用，进而可以找到该Binder引用对应的Binder实体。

```
uint32_t desc;
```

### binder\_proc

binder\_proc在binder.c中定义，它是Binder通信的进程在内核中的描述。

```
struct binder_proc {
    struct hlist_node proc_node;
    struct rb_root threads;
    struct rb_root nodes;　
    struct rb_root refs_by_desc;　
    struct rb_root refs_by_node;
    int pid;
    struct vm_area_struct *vma;
    struct mm_struct *vma_vm_mm;
    struct task_struct *tsk;
    struct files_struct *files;
    struct hlist_node deferred_work_node;
    int deferred_work;
    void *buffer;
    ptrdiff_t user_buffer_offset;

    struct list_head buffers;
    struct rb_root free_buffers;
    struct rb_root allocated_buffers;
    size_t free_async_space;

    struct page **pages;　
    size_t buffer_size;
    uint32_t buffer_free;
    struct list_head todo;　
    wait_queue_head_t wait;
    struct binder_stats stats;
    struct list_head delivered_death;
    int max_threads;　
    int requested_threads;
    int requested_threads_started;
    int ready_threads;
    long default_priority;
    struct dentry *debugfs_entry;
};
```

1. **struct hlist\_node proc\_node;** 全局哈希链表binder\_procs的节点，所有创建的binder\_proc都被加入到该链表
2. **struct rb\_root threads;**    binder\_thread红黑树，关联binder\_thread->rb\_node
3. **struct rb\_root nodes;**    binder\_node红黑树，关联binder\_node->rb\_node
4. **struct rb\_root refs\_by\_desc;**　以binder\_ref描述句柄为关键字的红黑树，关联binder\_ref->rb\_node\_desc
5. **struct rb\_root refs\_by\_node;**    以binder\_ref关联的实体地址为关键字的红黑树，关联binder\_ref->rb\_node
6. **int pid;**　进程id
7. **struct vm\_area\_struct \*vma;** 进程用户空间虚拟地址
8. **struct task\_struct \*tsk;**　进程控制块
9. **void \*buffer;**　进程内核空间虚拟地址
10. **ptrdiff\_t user\_buffer\_offset;**　内核虚拟地址和用户虚拟地址之间的偏移，通过该偏移可计算另一个虚拟空间地址
11. **struct list\_head todo;**　进程的待处理事务队列
12. **wait\_queue\_head\_t wait;**　线程等待队列
13. **max\_threads;**　最大线程数

### binder\_thread

binder\_thread在binder.c中定义，它是使用Binder通信的线程在内核中的描述。

```
struct binder_thread {
    struct binder_proc *proc;
    struct rb_node rb_node;　
    int pid;　
    int looper;　
    struct binder_transaction *transaction_stack;　
    struct list_head todo;　
    uint32_t return_error; /* Write failed, return error code in read buf */
    uint32_t return_error2; /* Write failed, return error code in read */
    wait_queue_head_t wait;
    struct binder_stats stats;
};

enum {
    BINDER_LOOPER_STATE_REGISTERED  = 0x01,
    BINDER_LOOPER_STATE_ENTERED     = 0x02,
    BINDER_LOOPER_STATE_EXITED      = 0x04,
    BINDER_LOOPER_STATE_INVALID     = 0x08,
    BINDER_LOOPER_STATE_WAITING     = 0x10,
    BINDER_LOOPER_STATE_NEED_RETURN = 0x20
};
```

1. **struct binder\_proc \*proc;**　指向所在进程的binder\_proc
2. **struct rb\_node rb\_node;**    关联到红黑树binder\_proc->threads
3. **int pid;**    进程id
4. **int looper;** 线程状态，可取的状态如上枚举所定义
5. **struct binder\_transaction \*transaction\_stack;** 正在处理的事务栈
6. **struct list\_head todo;** 待处理的事务链表
7. **wait\_queue\_head\_t wait;** 等待队列

### binder\_buffer

binder\_buffer在binder.c中定义，它用来描述Binder进程在内核中分配的内核缓冲区。

```
struct binder_buffer {
    struct list_head entry;
    struct rb_node rb_node;
    unsigned free:1;
    unsigned allow_user_free:1;
    unsigned async_transaction:1;
    unsigned debug_id:29;

    struct binder_transaction *transaction;

    struct binder_node *target_node;
    size_t data_size;
    size_t offsets_size;
    uint8_t data[0];
};
```

1. **struct list\_head entry;**　关联到binder\_proc->buffers链表，从而对内存进行管理
2. **struct rb\_node rb\_node;**    关联到binder\_proc->free\_buffers或binder\_proc->allocated\_buffers红黑树，从而对已有内存和空闲内存进行管理
3. **unsigned free:1;**　是否空闲
4. **struct binder\_transaction \*transaction;** 正在处理的事务
5. **struct binder\_node \*target\_node;** 正在使用该块内存的binder实体对象

### binder\_transaction

binder\_transaction用来描述Binder进程中通信过程，即通信中请求的事务。

```
struct binder_transaction {
    int debug_id;
    struct binder_work work;
    struct binder_thread *from;
    struct binder_transaction *from_parent;
    struct binder_proc *to_proc;
    struct binder_thread *to_thread;
    struct binder_transaction *to_parent;
    unsigned need_reply:1;
    /* unsigned is_dead:1; */    /* not used at the moment */

    struct binder_buffer *buffer;
    unsigned int    code;
    unsigned int    flags;
    long    priority;
    long    saved_priority;
    kuid_t    sender_euid;
};
```

1. **struct binder\_thread \*from;**　发起事务的源线程
2. **struct binder\_proc \*to\_proc;**    事务的目标进程
3. **struct binder\_thread \*to\_thread;**　事务的目标线程
4. **unsigned need\_reply:1;** 是否是同步事务，需对方回复
5. **struct binder\_buffer \*buffer;** 进程间通信的数据
6. **long    priority;** 源线程优先级
7. **kuid\_t    sender\_euid;** 源线程用户ID

### binder\_transaction\_data

binder\_transaction\_data用来描述Binder事务交互的数据格式。

```
struct binder_transaction_data {
    union {
        __u32    handle;　
        binder_uintptr_t ptr;
    } target;
    binder_uintptr_t    cookie;
    __u32        code;

    /* General information about the transaction. */
    __u32            flags;
    pid_t        sender_pid;
    uid_t        sender_euid;
    binder_size_t    data_size;    /* number of bytes of data */
    binder_size_t    offsets_size;    /* number of bytes of offsets */

    /* If this transaction is inline, the data immediately
     * follows here; otherwise, it ends with a pointer to
     * the data buffer.
     */
    union {
        struct {
            /* transaction data */
            binder_uintptr_t    buffer;
            /* offsets from buffer to flat_binder_object structs */
            binder_uintptr_t    offsets;
        } ptr;
        __u8    buf[8];
    } data;
};
```

1. **target**　当binder\_transaction\_data是由用户空间的进程发送给Binder驱动时，handle是该事务的发送目标在Binder驱动中的信息，即该事务会交给handle来处理，handle的值是目标在Binder驱动中的Binder引用。当binder\_transaction\_data是有Binder驱动反馈给用户空间进程时，ptr是该事务的发送目标在用户空间中的信息，即该事务会交给ptr对应的服务来处理，ptr是处理该事务的服务的服务在用户空间的本地Binder对象
2. **binder\_uintptr\_t    cookie;**    只有当事务是由Binder驱动传递给用户空间时，cookie才有意思，它的值是处理该事务的Server位于C++层的本地Binder对象
3. **\_\_u32　flags;**　事务编码。如果是请求，则以B&#x43;_&#x5F00;头；如果是回复，则以B&#x52;_&#x5F00;头。
4. **binder\_size\_t    data\_size;** 数据大小
5. **binder\_size\_t    offsets\_size;** 数据中包含的对象的个数
6. **data** 源线程优先级

### flat\_binder\_object

flat\_binder\_object描述进程中通信过程中传递的Binder实体/引用对象或文件描述符。

```
struct flat_binder_object {
    /* 8 bytes for large_flat_header. */
    __u32        type;
    __u32        flags;

    /* 8 bytes of data. */
    union {
        binder_uintptr_t    binder;    /* local object */
        __u32            handle;    /* remote object */
    };

    /* extra data associated with local object */
    binder_uintptr_t    cookie;
};
```

flat\_binder\_object通过type来区分描述类型，可选的描述类型如下：

```
enum {
  BINDER_TYPE_BINDER = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
  BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
  BINDER_TYPE_HANDLE = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
  BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
  BINDER_TYPE_FD = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```

type为BINDER\_TYPE\_BINDER，BINDER\_TYPE\_WEAK\_BINDER时，分别描述强类型实体对象与弱类型实体对象。type为BINDER\_TYPE\_HANDLE，BINDER\_TYPE\_WEAK\_HANDLE时，分别描述强类型引用对象与弱类型引用对象。type为BINDER\_TYPE\_FD时，描述文件描述符。 当描述实体对象时，cookie表示Binder实体对应的Service在用户空间的本地Binder(BBinder)地址，binder表示Binder实体对应的Service在用户空间的本地Binder(BBinder)的弱引用。 当描述引用对象时，handle表示该引用对象的句柄值。

## 2. 驱动代码分析

### binder\_init

binder\_init是binder驱动的初始化入口，在分析binder\_init之前，先看一下kernel include/linux/init.h中device\_initcall的实现。

```
#define device_initcall(fn)        __define_initcall(fn, 6)

//宏定义##表示字符串连接
#define __define_initcall(fn, id) \
    static initcall_t __initcall_##fn##id __used \
    __attribute__((__section__(".initcall" #id ".init"))) = fn

typedef int (*initcall_t)(void);

device_initcall(binder_init);
```

device\_initcall(binder\_init)即把binder\_init的函数指针\_\_initcall\_binder\_init6存放在initcall6.init section中，在kernel分级初始化时会调到该section的函数进行binder device初始化。

```
static const struct file_operations binder_fops = {
    .owner = THIS_MODULE,
    .poll = binder_poll,
    .unlocked_ioctl = binder_ioctl,
    .compat_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    .flush = binder_flush,
    .release = binder_release,
};

static struct miscdevice binder_miscdev = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "binder",
    .fops = &binder_fops
};

static int __init binder_init(void)
{
    int ret;

    binder_deferred_workqueue = create_singlethread_workqueue("binder");
    if (!binder_deferred_workqueue)
        return -ENOMEM;

    ......
    ret = misc_register(&binder_miscdev);
    ......
    return ret;
}
```

binder\_init主要是注册并创建misc设备节点/dev/binder，并关联binder\_fops到binder的file\_operations。

### binder\_open

用户空间系统调用open最终会调到binder驱动中的binder\_open。

```
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;
    .......
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;
    get_task_struct(current);　// 获取当前进程控制快current
    proc->tsk = current;
    INIT_LIST_HEAD(&proc->todo);　
    init_waitqueue_head(&proc->wait);
    proc->default_priority = task_nice(current);

    binder_lock(__func__);

    binder_stats_created(BINDER_STAT_PROC);
    hlist_add_head(&proc->proc_node, &binder_procs);
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    filp->private_data = proc;

    binder_unlock(__func__);
    .......
    return 0;
}
```

binder\_open创建了binder\_proc并加入到binder\_procs全局哈希链表binder\_procs。其中binder\_proc的task成员被初始化为当前进程的task\_struct current，并根据current初始化default\_priority，pid，初始化wait及todo队列。最后将创建的binder\_proc保存到filp->private\_data中，执行其他的file operations的操作时就可以根据filp指针获取binder\_proc。

### binder\_mmap

binder\_mmap主要完成binder进程用户虚拟地址空间与内核虚拟地址空间的映射与分配。

```
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;

    struct vm_struct *area;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;

    if (proc->tsk != current)
        return -EINVAL;

    // 最多分配４M缓存区
    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;

    // 分配的缓存区只可读，不可写也不能拷贝
    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
        ret = -EPERM;
        failure_string = "bad vm_flags";
        goto err_bad_arg;
    }
    vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

    mutex_lock(&binder_mmap_lock);
    if (proc->buffer) {
        ret = -EBUSY;
        failure_string = "already mapped";
        goto err_already_mapped;
    }
    // 在内核地址空间分配一块vma->vm_end - vma->vm_start大小的空间
    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
    if (area == NULL) {
        ret = -ENOMEM;
        failure_string = "get_vm_area";
        goto err_get_vm_area_failed;
    }
    // 内核空间缓存地址proc->buffer，用户空间地址vma->vm_start，proc->user_buffer_offset是二个地址间的偏移
    proc->buffer = area->addr;
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
    mutex_unlock(&binder_mmap_lock);

    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    if (proc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    proc->buffer_size = vma->vm_end - vma->vm_start;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;

    /* binder_update_page_range assumes preemption is disabled */
    preempt_disable();
    // 为用户/内核虚拟地址空间分配物理页面并映射
    ret = binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma);
    preempt_enable_no_resched();
    if (ret) {
        ret = -ENOMEM;
        failure_string = "alloc small buf";
        goto err_alloc_small_buf_failed;
    }
    buffer = proc->buffer;
    // 分配的binder_buffer加入到proc->buffers进行管理
    INIT_LIST_HEAD(&proc->buffers);
    list_add(&buffer->entry, &proc->buffers);
    buffer->free = 1;
    binder_insert_free_buffer(proc, buffer);
    proc->free_async_space = proc->buffer_size / 2;
    barrier();
    proc->files = get_files_struct(current);
    proc->vma = vma;
    proc->vma_vm_mm = vma->vm_mm;

    return 0;
    ......
}
```

1. 首先判断proc->tsk是否对应当前进程，不是则退出．然后限制mmap的用户空间vm\_area\_struct最大为**４M**,即**biner驱动最多分配４M内核空间来进行进程间通信**
2. 设置vma->vm\_flags标志为只读，即分配的空间不可写
3. 根据proc->buffer是否为null判断是否重复mmap,然后调用get\_vm\_area根据vma来分配内核空间地址，并设置起始地址proc->buffer，与用户空间地址偏移proc->user\_buffer\_offset
4. 调用binder\_update\_page\_rang为用户/内核虚拟地址空间分配对应的物理页面，**同一块物理内存分别映射到内核虚拟地址空间和用户虚拟内存空间**
5. 分配内存后，初始化binder\_buffer相关数据结构

### binder\_ioctl

binder\_ioctl支持的命令在binder.h中定义，如下：

```
#define BINDER_WRITE_READ        _IOWR('b', 1, struct binder_write_read)　//收发Binder IPC数据
#define    BINDER_SET_IDLE_TIMEOUT        _IOW('b', 3, __s64)    //Not Used
#define    BINDER_SET_MAX_THREADS        _IOW('b', 5, __u32)    //设置Binder线程最大个数
#define    BINDER_SET_IDLE_PRIORITY    _IOW('b', 6, __s32) //Not Used
#define    BINDER_SET_CONTEXT_MGR        _IOW('b', 7, __s32)    //设置Service Manager
#define    BINDER_THREAD_EXIT        _IOW('b', 8, __s32) //Not Used
#define BINDER_VERSION            _IOWR('b', 9, struct binder_version)    //获取Binder版本信息
```

binder\_ioctl的实现在后面根据具体的业务逻辑具体分析。
