# Android Binder之概述

## Binder简介

在Android系统中，应用程序是由组件(Acyiviy/Service/Broadcast Receiver/Content Provider)组成的，组件可以运行在一个进程中，也可以分别运行在不同进程中。同样，在Android系统中的各类系统服务也可以运行在一个或多个进程中，运行在不同进程的应用程序组件或系统服务之间的通信都是通过Binder来进行的。

由于Android是基于Linux内核开发的，在Linux系统中传统的IPC机制有**Pipe**，**ShareMemory**，**Singal**，**Message**，**Socket**等，但Android并未完全采用这些传统的进程间通信机制，而是引入了Binder作为其主要的IPC机制。

Binder是在[OpenBinder](http://www.angryredplanet.com/~hackbod/openbinder/docs/html/BinderIPCMechanism.html)的基础上实现的，OpenBinder是一套开源的系统IPC机制，最初是由Be Inc开发，后来由Palm, Inc公司负责开发，Google对其改造后应用到了Android系统上。

Binder相比其他的IPC机制有什么优势?&#x20;

* **基于Client-Server的通信方式。**&#x41;PP通过调用Service接口就可以非常方便的完成很多功能。&#x20;
* **高效率**。Binder中数据传输只需要一次拷贝(只需将数据通过驱动发送到目标进程的内核虚拟缓存区，而使用Binder通信用户空间缓存区与内核空间的缓存区映射到同一物理内存的，所以只需要一次拷贝)，节省时间的同时也节省了空间。&#x20;
* **面向对象的RPC调用，模糊了进程边界，淡化进程间通信过程**。在Android的C++/java面向对象语言环境中，Binder更加符合面向对象思想，Binder的实体位于一个进程中，而引用却可以分布在各个进程中，通过Binder引用调用就像本地调用一样，这种通信方式更加适合Android组件化的思想。&#x20;
* **安全性**。Android为每个安装的应用程序分配了UID，Binder能依据调用进程的UID/PID来进行权限控制，而传统IPC需要上层协议来验证权限保证安全性。

## Binder机制

Binder采用的是CS通信方式，对于提供服务的进程即Server进程，请求服务的进程即Client进程。同一个Server进程可能存在多个Service组件，同时同一个Clent进程可以请求多个Service组件。 如下图所示，Binder的通信涉及到4个角色，分别是Client，Service，Service Mananger与Binder Driver。

![](<../../.gitbook/assets/image (386).png>)

Client，Service，Service Mananger都运行在用户空间独立的进程中，由于用户进程地址空间是独立且不能相互访问，而内核空间却是是全局共享，所以可以通过内核空间的缓冲区进行数据的中转。如下图示Client，Service，Service Mananger通过SystemCall(open，mmap，ioctl)访问/dev/binder，通过Binder driver间接建立与用户空间其他进程的通信通道。

![](<../../.gitbook/assets/image (312).png>)

## Binder分析

* Android Binder之概述
* Android Binder之Binder Driver
* Android Binder之Service Manager
* Android Binder之进程间通信库
* Android Binder之Service Manager代理对象
* Android Binder之Service启动
* Android Binder之Service代理获取
* Android Binder之Java接口

## 参考资料

* [Android Binder设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)
* [Android系统源代码情景分析](http://0xcc0xcd.com/p/index.php)
