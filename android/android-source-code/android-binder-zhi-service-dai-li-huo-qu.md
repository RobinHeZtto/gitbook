# Android Binder之Service代理获取

> Service组件注册到Service Manager后，就在Server进程中等待Client的请求，而在Client请求之前，必须得先通过ServiceManager获取到Service的代理，才能与Service进行通信。本篇继续以MediaPlayerService为例，分析获取MediaPlayerService的代理对象的流程。相关代码在以下文件中: frameworks/av/media/libmedia/IMediaDeathNotifier.cpp frameworks/native/libs/binder/IServiceManager.cpp

根据前面Binder的分析，获取MediaPlayerService的代理对象，首先需要从驱动中拿到MediaPlayerService对应的binder\_ref的句柄值，根据句柄值创建Binder代理对象BpBinder，然后根据BpBinder封装成BpMediaPlayerService。下面从MediaPlayer的getService入口开始分析。

```
>>> frameworks/av/media/libmedia/IMediaDeathNotifier.cpp

sp<IMediaPlayerService> IMediaDeathNotifier::sMediaPlayerService;

IMediaDeathNotifier::getMediaPlayerService()
{
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService == 0) {
        sp<IServiceManager> sm = defaultServiceManager();
        sp<IBinder> binder;
        do {
            binder = sm->getService(String16("media.player"));
            if (binder != 0) {
                break;
            }
            usleep(500000); // 0.5 s
        } while (true);

        if (sDeathNotifier == NULL) {
            sDeathNotifier = new DeathNotifier();
        }
        binder->linkToDeath(sDeathNotifier);
        sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    return sMediaPlayerService;
}
```

首先通过defaultServiceManager()获取到ServiceManager的代理，然后通过ServiceManager代理的成员方法getService获取MediaPlayerService的Bpbinder对象，通过interface\_cast将BpBinder封装成BpMediaPlayerService并获取其IMediaPlayerService接口。defaultServiceManager()->getService的实现如下：

```
>>> frameworks/native/libs/binder/IServiceManager.cpp

virtual sp<IBinder> getService(const String16& name) const
{
    unsigned n;
    for (n = 0; n < 5; n++){
        sp<IBinder> svc = checkService(name);
        if (svc != NULL) return svc;
        ALOGI("Waiting for service %s...\n", String8(name).string());
        sleep(1);
    }
    return NULL;
}

virtual sp<IBinder> checkService( const String16& name) const
{
    Parcel data, reply;
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    data.writeString16(name);
    remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
    return reply.readStrongBinder();
}
```

getService()中通过checkService获取name为"media.player"的BpBinder对象，获取失败则休眠1s再次循环获取，循环超过5次则获取失败退出。checkService的操作与前篇分析的addService操作类似，将进程间通信数据封装成Parcel对象并调用remote()->transact()将数据发送给驱动，接下来的数据发送的过程与前篇addService流程一致。 下面看一下ServiceManager端的处理。

```
>>> frameworks/native/cmds/servicemanager/service_manager.c

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

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;
    ......
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

svcmgr\_handler()中首先获取到需要查找的服务名称media.player然后通过do\_find\_service()查找对应的服务，并通过bio\_put\_ref()将handle返回给驱动。

```
>>> frameworks/native/cmds/servicemanager/service_manager.c

uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    struct svcinfo *si = find_svc(s, len);

    if (!si || !si->handle) {
        return 0;
    }

    ......

    if (!svc_can_find(s, len, spid, uid)) {
        return 0;
    }

    return si->handle;
}
```

do\_find\_service()在svclist中找到对应的svcinfo，并返回其句柄值handle。在ServiceManager将MediaPlayerService引用的句柄返回给驱动后，驱动会根据handle查找Client进程是否已经存在MediaPlayerService的binder\_ref对象，如果不存在则为其创建该引用对象，并将该引用对象的句柄值返回给Client，根据此handle值就可以封装对应MediaPlayerService的Bpbinder及代理对象。
