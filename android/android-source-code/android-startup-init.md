# Android启动之init

## 1. 概述

Init进程是Kernel启动后在用户空间创建的第一个用户级进程，它的pid为1。其他所有的用户进程都是由其直接或间接fork产生的.

![](<../../.gitbook/assets/image (204).png>)

Init调用流程如下图示,从汇编代码kernel/arch/arm/kernel/head-common.S `b start_kernel`进入到C环境kernel/init/main.c `start_kernel`执行.最后在run\_init\_process()中通过do\_execve创建Init进程.​

![](<../../.gitbook/assets/image (70).png>)

kernel\_init()中的execute\_command即/init，run\_init\_process通过execve()系统调用来启动init进程。如果没有定义execute\_command，则在/sbin，/etc，/bin查找，否则Kernel Panic.

```
static int __ref kernel_init(void *unused)
{
  ......
  if (execute_command) {
    ret = run_init_process(execute_command);
    if (!ret)
      return 0;
    pr_err("Failed to execute %s (error %d).  Attempting defaults...\n",
      execute_command, ret);
  }
  if (!try_to_run_init_process("/sbin/init") ||
      !try_to_run_init_process("/etc/init") ||
      !try_to_run_init_process("/bin/init") ||
      !try_to_run_init_process("/bin/sh"))
    return 0;
​
  panic("No working init found.  Try passing init= option to kernel. "
        "See Linux Documentation/init.txt for guidance.");
}
```

Android中的Init进程与Linux中有所不同,它主要实现以下的四大功能：

* 解析执行init.rc文件
* 创建设备节点
* 创建关键的daemon进程并处理子进程的终止
* 属性服务

## 2. 执行流程

> Init的源码主要集中在system/core/init/

下面从init.cpp的main()开始逐段来分析init进程的执行流程:

```
//根据执行的文件名argv[0]判断是否是执行ueventd
if (!strcmp(basename(argv[0]), "ueventd")) {
    return ueventd_main(argc, argv);
}
//根据执行的文件名argv[0]判断是否是执行watchdogd
if (!strcmp(basename(argv[0]), "watchdogd")) {
    return watchdogd_main(argc, argv);
}
​
// Clear the umask.
umask(0);
​
// 初始化PATH环境变量 PATH=/sbin:/vendor/bin:/system/sbin:/system/bin:/system/xbin
add_environment("PATH", _PATH_DEFPATH);　
​
// 根据argc，argv[1]判断是否是first_stage（first_stage运行在kernel domain）.
bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);
​
// mount并创建相关内存文件系统及目录
if (is_first_stage) {
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    #define MAKE_STR(x) __STRING(x)
    mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
    mount("sysfs", "/sys", "sysfs", 0, NULL);
}
```

1. 首先根据执行参数basename(argv\[0])判断是否是执行ueventd或watchdogd(ueventd与watchdogd功能代码在init中实现),通过system/core/init/Android.mk可以知道,/sbin/ueventd与/sbin/watchdogd实际上都链接到init.

```
# Create symlinks
LOCAL_POST_INSTALL_CMD := $(hide) mkdir -p $(TARGET_ROOT_OUT)/sbin; \
    ln -sf ../init $(TARGET_ROOT_OUT)/sbin/ueventd; \
    ln -sf ../init $(TARGET_ROOT_OUT)/sbin/watchdogd
```

1. umask(0) 即设置创建文件的属性默认为0777.
2. is\_first\_stage是用来判断init是在内核空间还是在用户空间启动,is\_first\_stage为true意味着init是运行在kernel domain，因为selinux的相关设置需要在内核空间下.
3. 如果是在内核启动,创建/dev，/proc，/sys目录并mount相应的虚拟内存文件系统

```
open_devnull_stdio();
klog_init();
klog_set_level(KLOG_NOTICE_LEVEL);
```

1. open\_devnull\_stdio();屏蔽标准输入/输出/错误,即无法通过stdin/stdout/stderr输出，如下代码所示,通过dup2()复制文件描述符，重定向stdin,stdout,stderr到/sys/fs/selinux/null或/dev/\__null\__&#x4E0A;.(一般daemon进程都会有类似的屏蔽操作)

```
# ls /proc/1/fd -al
lrwx------ 1 root root 64 1970-01-04 07:34 0 -> /sys/fs/selinux/null
lrwx------ 1 root root 64 1970-01-04 07:34 1 -> /sys/fs/selinux/null
lrwx------ 1 root root 64 1970-01-04 07:34 2 -> /sys/fs/selinux/null
​
void open_devnull_stdio(void)
{
    // Try to avoid the mknod() call if we can. Since SELinux makes
    // a /dev/null replacement available for free, let's use it.
    int fd = open("/sys/fs/selinux/null", O_RDWR);
    if (fd == -1) {
        // OOPS, /sys/fs/selinux/null isn't available, likely because
        // /sys/fs/selinux isn't mounted. Fall back to mknod.
        static const char *name = "/dev/__null__";
        if (mknod(name, S_IFCHR | 0600, (1 << 8) | 3) == 0) {
            fd = open(name, O_RDWR);
            unlink(name);
        }
        if (fd == -1) {
            exit(1);
        }
    }
​
    dup2(fd, 0);
    dup2(fd, 1);
    dup2(fd, 2);
    if (fd > 2) {
        close(fd);
    }
}
```

1. klog\_init,创建kmsg设备节点,printk打印的log可以通过cat /proc/kmsg或者dmesg输出.

```
void klog_init(void) {
    if (klog_fd >= 0) return; /* Already initialized */
​
    klog_fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    if (klog_fd >= 0) {
        return;
    }
​
    static const char* name = "/dev/__kmsg__";
    if (mknod(name, S_IFCHR | 0600, (1 << 8) | 11) == 0) {
        klog_fd = open(name, O_WRONLY | O_CLOEXEC);
        unlink(name);
    }
}
```

1. klog\_set\_level设置log等级

```
if (!is_first_stage) {
    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
​
    property_init();
​
    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();
​
    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();
}
```

1. 创建/dev/.booting，用来标志是否处于初始化过程中。如下init.rc代码片段所示，在late-init阶段，将触发firmware\_mounts\_complete，删除掉/dev/.booting

```
# Indicate to fw loaders that the relevant mounts are up.
on firmware_mounts_complete
    rm /dev/.booting
​
# Mount filesystems and start core system services.
on late-init
    ......
    # Remove a file to wake up anything waiting for firmware.
    trigger firmware_mounts_complete
    .......
```

1. 调用property\_init()初始化属性系统，在后面属性服务中详细描述.
2. 调用process\_kernel\_dt()处理设置ro.boot.android,ro.boot.firmware二个属性.

```
static void process_kernel_dt() {
    static const char android_dir[] = "/proc/device-tree/firmware/android";
​
    std::string file_name = android::base::StringPrintf("%s/compatible", android_dir);
​
    std::string dt_file;
    android::base::ReadFileToString(file_name, &dt_file);
    if (!dt_file.compare("android,firmware")) {
        ERROR("firmware/android is not compatible with 'android,firmware'\n");
        return;
    }
​
    std::unique_ptr<DIR, int(*)(DIR*)>dir(opendir(android_dir), closedir);
    if (!dir) return;
​
    struct dirent *dp;
    while ((dp = readdir(dir.get())) != NULL) {
        if (dp->d_type != DT_REG || !strcmp(dp->d_name, "compatible") || !strcmp(dp->d_name, "name")) {
            continue;
        }
​
        file_name = android::base::StringPrintf("%s/%s", android_dir, dp->d_name);
​
        android::base::ReadFileToString(file_name, &dt_file);
        std::replace(dt_file.begin(), dt_file.end(), ',', '.');
​
        std::string property_name = android::base::StringPrintf("ro.boot.%s", dp->d_name);
        property_set(property_name.c_str(), dt_file.c_str());
    }
}
```

1.  调用process\_kernel\_cmdline()读取/proc/cmdline文件,并设置cmdline文件中以androidboot.开头对应ro.boot.的属性值.例如`androidboot.hardware=qcom -> [ro.boot.hardware]: [qcom]`

    ```
    static void process_kernel_cmdline() {
        // Don't expose the raw commandline to unprivileged processes.
        chmod("/proc/cmdline", 0440);
    ​
        // The first pass does the common stuff, and finds if we are in qemu.
        // The second pass is only necessary for qemu to export all kernel params
        // as properties.
        import_kernel_cmdline(false, import_kernel_nv);
        if (qemu[0]) import_kernel_cmdline(true, import_kernel_nv);
    }
    ​
    static void import_kernel_nv(const std::string& key, const std::string& value, bool for_emulator) {
        if (key.empty()) return;
    ​
        if (for_emulator) {
            // In the emulator, export any kernel option with the "ro.kernel." prefix.
            property_set(android::base::StringPrintf("ro.kernel.%s", key.c_str()).c_str(), value.c_str());
            return;
        }
    ​
        if (key == "qemu") {
            strlcpy(qemu, value.c_str(), sizeof(qemu));
        } else if (android::base::StartsWith(key, "androidboot.")) {
            property_set(android::base::StringPrintf("ro.boot.%s", key.c_str() + 12).c_str(),
                         value.c_str());
        }
    }
    ```

    1. 调用export\_kernel\_boot\_props设置boot相关的属性.即根据设置prop\_map中src\_prop设置dst\_prop的值.

    ```
    static void export_kernel_boot_props() {
        struct {
            const char *src_prop;
            const char *dst_prop;
            const char *default_value;
        } prop_map[] = {
            { "ro.boot.serialno",   "ro.serialno",   "", },
            { "ro.boot.mode",       "ro.bootmode",   "unknown", },
            { "ro.boot.baseband",   "ro.baseband",   "unknown", },
            { "ro.boot.bootloader", "ro.bootloader", "unknown", },
            { "ro.boot.hardware",   "ro.hardware",   "unknown", },
            { "ro.boot.revision",   "ro.revision",   "0", },
        };
        for (size_t i = 0; i < ARRAY_SIZE(prop_map); i++) {
            std::string value = property_get(prop_map[i].src_prop);
            property_set(prop_map[i].dst_prop, (!value.empty()) ? value.c_str() : prop_map[i].default_value);
        }
    }
    ```

    ```
    selinux_initialize(is_first_stage);
    ​
    // If we're in the kernel domain, re-exec init to transition to the init domain now
    // that the SELinux policy has been loaded.
    if (is_first_stage) {
        if (restorecon("/init") == -1) {
            ERROR("restorecon failed: %s\n", strerror(errno));
            security_failure();
        }
        char* path = argv[0];
        char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
        // 初始化selinux后，在user domain执行init --second-stage
        if (execv(path, args) == -1) {
            ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
            security_failure();
        }
    }
    ​
    NOTICE("Running restorecon...\n");
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon("/property_contexts");
    restorecon_recursive("/sys");
    ```

    在kernel domaind调用selinux\_initialize对selinux进行初始化,然后通过execv重新执行init --second-stage。

    ```
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        ERROR("epoll_create1 failed: %s\n", strerror(errno));
        exit(1);
    }
    ​
    signal_handler_init();
    ​
    property_load_boot_defaults();
    export_oem_lock_status();
    start_property_service();
    ```

    1. epoll\_create1创建epoll实例处理子进程退出及属性服务事件（EPOLL\_CLOEXEC标志与open的O\_CLOEXEC标志类似，即进程被替换时会关闭文件描述符），epoll\_fd对应关联/proc/1/fd/下`eventpoll`的fd。
    2. signal\_handler\_init()子进程信号处理,后面单独分析
    3. 初始化default属性并启动服务系统,后面单独分析

    ```
        const BuiltinFunctionMap function_map;
        Action::set_function_map(&function_map);
    ​
        Parser& parser = Parser::GetInstance();
        //ServiceParser解析`service section`
        parser.AddSectionParser("service",std::make_unique<ServiceParser>());
        //ActionParser解析`action section`
        parser.AddSectionParser("on", std::make_unique<ActionParser>());
        //ImportParser解析`import rc`
        parser.AddSectionParser("import", std::make_unique<ImportParser>());
        //开始解析init.rc
        parser.ParseConfig("/init.rc");
    ​
        ActionManager& am = ActionManager::GetInstance();
    ​
        am.QueueEventTrigger("early-init");
    ​
        // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
        am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
        // ... so that we can start queuing up actions that require stuff from /dev.
        am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
        am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
        am.QueueBuiltinAction(keychord_init_action, "keychord_init");
        am.QueueBuiltinAction(console_init_action, "console_init");
    ​
        // Trigger all the boot actions to get us started.
        am.QueueEventTrigger("init");
    ​
        // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
        // wasn't ready immediately after wait_for_coldboot_done
        am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    ​
        // Don't mount filesystems or start core system services in charger mode.
        std::string bootmode = property_get("ro.bootmode");
        if (bootmode == "charger") {　 // 是否是关机充电模式
            am.QueueEventTrigger("charger");
        } else {
            am.QueueEventTrigger("late-init");
        }
    ​
        // Run all property triggers based on current state of the properties.
        am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
    ```

    1. 根据action,service,import解析/init.rc,其中ServiceParser，ActionParser，ImportParser都继承自SectionParse，根据不同的keryword解析initrc文件，将service与action项分别加入到servic&#x65;_&#x4E0E;actio&#x6E;_&#x8868;中。
    2. 通过QueueEventTrigger，QueueBuiltinAction把对应的action放到trigger\_queue\_表中。

    ```
    while (true) {
        //判断是否有事件需要处理
        if (!waiting_for_exec) {
            //依次执行每个action中command
            am.ExecuteOneCommand();
            //重启退出的进程
            restart_processes();
        }
    ​
        int timeout = -1;
        //有进程需要重启则等待该进程重启
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }
    ​
        //还有action要处理则不等待
        if (am.HasMoreCommands()) {
            timeout = 0;
        }
    ​
        bootchart_sample(&timeout);
    ​
        epoll_event ev;
        //等待epoll_event事件到来，等待timeout时间
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            //epoll_event事件到来，执行对应处理函数
            ((void (*)()) ev.data.ptr)();
        }
    }
    ```

    while循环不断调用ExecuteOneCommand函数时，将按照trigger表的顺序，依次取出action链表中与trigger匹配的action。每次均执行一个action中的一个command对应函数（一个action可能携带多个command）。当一个action所有的command均执行完毕后，再执行下一个action。当一个trigger对应的action均执行完毕后，再执行下一个trigger对应action。

    **子进程信号处理**

    由于Init进程是系统的1号进程，其他用户进程都是由其直接或间接生成，所以init进程的作用之一就是处理子进程的退出。进程在退出时内核会发出SIGCHLD信号，父进程收到该信号就可以处理子进程的退出从而防止子进程变成僵尸进程。另外，当进程退出时，该进程下所有的子进程都将变成init的子进程，由init负责对他们退出的处理。下面从signal\_handler\_init()分析Init对子进程退出的处理。

    ```
    void signal_handler_init() {
        int s[2];
        // 创建一对连接的socket
        if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s) == -1) {
            ERROR("socketpair failed: %s\n", strerror(errno));
            exit(1);
        }
    ​
        signal_write_fd = s[0];
        signal_read_fd = s[1];
    ​
        // 注册SIGCHLD的处理函数
        struct sigaction act;
        memset(&act, 0, sizeof(act));
        act.sa_handler = SIGCHLD_handler;
        act.sa_flags = SA_NOCLDSTOP;
        sigaction(SIGCHLD, &act, 0);
    ​
        // 处理函数，处理子进程的退出
        ServiceManager::GetInstance().ReapAnyOutstandingChildren();
    ​
        // 将signal_read_fd注册到epoll_fd上的的epoll可读事件
        register_epoll_handler(signal_read_fd, handle_signal);
    }
    ```

    1. 为了同时处理多个信号，通过socket来通信，socketpair创建一对已连接的本地socket，signal\_write\_fd端写，signal\_read\_fd端读。
    2. sigaction注册SIGCHLD信号的处理函数SIGCHLD\_handler(SA\_NOCLDSTOP标志只有当子进程结束才接收SIGCHLD信号)，即收到SIGCHLD信号后往signal\_write\_fd中写入"1"，触发signal\_read\_fd上的epoll处理事件handle\_signal

    ```
    static void SIGCHLD_handler(int) {
        if (TEMP_FAILURE_RETRY(write(signal_write_fd, "1", 1)) == -1) {
            ERROR("write(signal_write_fd) failed: %s\n", strerror(errno));
        }
    }
    ```

    1. ReapAnyOutstandingChildren()处理子进程的退出，下面具体分析
    2. register\_epoll\_handler注册signal\_read\_fd到epoll\_fd上，并关联EPOLLIN读事件，epoll\_fd上的事件在main()中循环处理

    ```
    void register_epoll_handler(int fd, void (*fn)()) {
        epoll_event ev;
        ev.events = EPOLLIN;
        ev.data.ptr = reinterpret_cast<void*>(fn);
        if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev) == -1) {
            ERROR("epoll_ctl failed: %s\n", strerror(errno));
        }
    }
    ​
    static void handle_signal() {
        // Clear outstanding requests.
        char buf[32];
        read(signal_read_fd, buf, sizeof(buf));
    ​
        ServiceManager::GetInstance().ReapAnyOutstandingChildren();
    }
    ```

    最终，在ServiceManager的ReapAnyOutstandingChildren()来处理子进程的退出.下面具体分析ReapAnyOutstandingChildren

    ```
    void ServiceManager::ReapAnyOutstandingChildren() {
        while (ReapOneProcess()) {
        }
    }
    ​
    ​
    bool ServiceManager::ReapOneProcess() {
        int status;
        // 等待子进程结束避免成为僵尸进程(第一个参数-1表示等待任意子进程结束，最后一个参数WNOHANG表示非阻塞)
        pid_t pid = TEMP_FAILURE_RETRY(waitpid(-1, &status, WNOHANG));
        // 无子进程退出，返回false结束循环
        if (pid == 0) {
            return false;
            //执行失败，返回false结束循环
        } else if (pid == -1) {
            ERROR("waitpid failed: %s\n", strerror(errno));
            return false;
        }
    ​
        // 根据pid在services_中查找对应的service
        Service* svc = FindServiceByPid(pid);
    ​
        std::string name;
        if (svc) {
            name = android::base::StringPrintf("Service '%s' (pid %d)",
                                               svc->name().c_str(), pid);
        } else {
            name = android::base::StringPrintf("Untracked pid %d", pid);
        }
    ​
        // 输出相关信息
        if (WIFEXITED(status)) {
            NOTICE("%s exited with status %d\n", name.c_str(), WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            NOTICE("%s killed by signal %d\n", name.c_str(), WTERMSIG(status));
        } else if (WIFSTOPPED(status)) {
            NOTICE("%s stopped by signal %d\n", name.c_str(), WSTOPSIG(status));
        } else {
            NOTICE("%s state changed", name.c_str());
        }
    ​
        // 不是服务进程，处理到此结束，直接退出
        if (!svc) {
            return true;
        }
    ​
        // 服务进程，svc->Reap()进一步处理
        if (svc->Reap()) {
            waiting_for_exec = false;
            RemoveService(*svc);
        }
    ​
        return true;
    }
    ```

    1. while循环中调用ReapOneProcess处理
    2. waitpid()等待子进程退出结束(WNOHANG标志没有子进程死亡立即返回)，以获取子进程结束信息清除zombie
    3. 对于一般子进程,waitpid后处理结束.但如果退出的是服务进程的话,那么会通过svc->Reap()对其进一步处理

    ```
    bool Service::Reap() {
       //kill未定义SVC_ONESHOT或定义了SVC_RESTART标志的service的子进程
       if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {
           NOTICE("Service '%s' (pid %d) killing any children in process group\n",
                  name_.c_str(), pid_);
            kill(-pid_, SIGKILL);
        }
    ​
        // 清除创建的socket
        for (const auto& si : sockets_) {
            std::string tmp = StringPrintf(ANDROID_SOCKET_DIR "/%s", si.name.c_str());
            unlink(tmp.c_str());
        }
    ​
        if (flags_ & SVC_EXEC) {
            INFO("SVC_EXEC pid %d finished...\n", pid_);
            return true;
        }
    ​
        pid_ = 0;
        flags_ &= (~SVC_RUNNING);
    ​
        // Oneshot processes go into the disabled state on exit,
        //未定义SVC_ONESHOT与SVC_RESTART的service，标志置为SVC_DISABLED，不再启动
        if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART)) {
            flags_ |= SVC_DISABLED;
        }
    ​
        // Disabled and reset processes do not get restarted automatically.
        if (flags_ & (SVC_DISABLED | SVC_RESET))  {
            NotifyStateChange("stopped");
            return false;
        }
    ​
        time_t now = gettime();
        // 定义SVC_CRITICAL且没有定义SVC_RESTART的service重启超过４次进入recovery
        if ((flags_ & SVC_CRITICAL) && !(flags_ & SVC_RESTART)) {
            if (time_crashed_ + CRITICAL_CRASH_WINDOW >= now) {
                if (++nr_crashed_ > CRITICAL_CRASH_THRESHOLD) {
                    ERROR("critical process '%s' exited %d times in %d minutes; "
                          "rebooting into recovery mode\n", name_.c_str(),
                          CRITICAL_CRASH_THRESHOLD, CRITICAL_CRASH_WINDOW / 60);
                    android_reboot(ANDROID_RB_RESTART2, 0, "recovery");
                    return false;
                }
            } else {
                time_crashed_ = now;
                nr_crashed_ = 1;
            }
        }
    ​
        flags_ &= (~SVC_RESTART);
        flags_ |= SVC_RESTARTING;
    ​
        //执行当前service中所有onrestart命令
        onrestart_.ExecuteAllCommands();
    ​
        NotifyStateChange("restarting");
        return false;
    }
    ```

    Reap()中主要是对service进程的flag进行分析,来判断是否需要重启该service进程,定义了SVC\_ONESHOT的服务进程不会重启,状态改为stopped.定义了SVC\_CRITICAL与SVC\_RESTART的关键服务如果重启超过4次,系统将reboot到recovery.服务进程在重启前将移除其创建的socket(/dev/socket/),杀死其所有子进程,并通过NotifyStateChange改变其状态为restarting(即设置init.svc.属性状态),并执行initrc中该服务定义的onrestart命令.

    **属性服务**

    Android的属性是以字符串键值形式保存的系统的关键值(可通过adb shell getprop打印所有的prop),在Android系统中，很多系统模块/应用的功能都是通过属性来控制的,属性设置也是通过Init进行管理管理理. 下面从void property\_init开始分析属性服务：

    ```
    // system/core/init/property_service.cpp
    // property_init创建一块用于存储属性的共享内存
    void property_init() {
        if (__system_property_area_init()) {
            ERROR("Failed to initialize property area\n");
            exit(1);
        }
    }
    ​
    // bionic/libc/include/sys/_system_properties.h
    ​
    #define PROP_FILENAME "/dev/__properties__"
    ​
    /*
    ** Initialize the area to be used to store properties.  Can
    ** only be done by a single process that has write access to
    ** the property area.
    */
    int __system_property_area_init();
    ​
    // bionic/libc/bionic/system_properties.cpp
    ​
    static char property_filename[PROP_FILENAME_MAX] = PROP_FILENAME;
    ​
    int __system_property_area_init()
    {
        free_and_unmap_contexts();
        mkdir(property_filename, S_IRWXU | S_IRGRP | S_IXGRP | S_IROTH | S_IXOTH);
        if (!initialize_properties()) {
            return -1;
        }
        bool open_failed = false;
        bool fsetxattr_failed = false;
        list_foreach(contexts, [&fsetxattr_failed, &open_failed](context_node* l) {
            if (!l->open(true, &fsetxattr_failed)) {
                open_failed = true;
            }
        });
        if (open_failed || !map_system_property_area(true, &fsetxattr_failed)) {
            free_and_unmap_contexts();
            return -1;
        }
        initialized = true;
        return fsetxattr_failed ? -2 : 0;
    }
    ​
    int __system_property_area_init()
    {
        return map_prop_area_rw();
    }
    ​
    static int map_prop_area_rw()
    {
        /* dev is a tmpfs that we can use to carve a shared workspace
         * out of, so let's do that...
         */
        const int fd = open(property_filename,
                            O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC | O_EXCL, 0444);
    ​
        if (fd < 0) {
            if (errno == EACCES) {
                /* for consistency with the case where the process has already
                 * mapped the page in and segfaults when trying to write to it
                 */
                abort();
            }
            return -1;
        }
    ​
        if (ftruncate(fd, PA_SIZE) < 0) {
            close(fd);
            return -1;
        }
    ​
        pa_size = PA_SIZE;
        pa_data_size = pa_size - sizeof(prop_area);
        compat_mode = false;
    ​
        void *const memory_area = mmap(NULL, pa_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
        if (memory_area == MAP_FAILED) {
            close(fd);
            return -1;
        }
    ​
        prop_area *pa = new(memory_area) prop_area(PROP_AREA_MAGIC, PROP_AREA_VERSION);
    ​
        /* plug into the lib property services */
        __system_property_area__ = pa;
    ​
        close(fd);
        return 0;
    }
    ```

    property\_init主要是分配一块共享的内存区域(/dev/**properties**)存储属性值 //TODO

    ```
    ./bionic/libc/include/sys/_system_properties.h:90:#define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"
    ​
    property_load_boot_defaults();
    ```

    property\_load\_boot\_defaults中读取并初始化/default.prop文件中的属性值,接着调用start\_property\_service()启动属性服务

    ```
    void start_property_service() {
        property_set_fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                        0666, 0, 0, NULL);
        if (property_set_fd == -1) {
            ERROR("start_property_service socket creation failed: %s\n", strerror(errno));
            exit(1);
        }
    ​
        listen(property_set_fd, 8);
    ​
        register_epoll_handler(property_set_fd, handle_property_set_fd);
    }
    ```

    start\_property\_service中创建并监听属性服务的socket，并加入到property\_set\_fd上的epoll事件监听中,其他进程属性设置的请求都将通过handle\_property\_set\_fd来处理。

    ```
    static void handle_property_set_fd()
    {
        prop_msg msg;
        int s;
        int r;
        struct ucred cr;
        struct sockaddr_un addr;
        socklen_t addr_size = sizeof(addr);
        socklen_t cr_size = sizeof(cr);
        char * source_ctx = NULL;
        struct pollfd ufds[1];
        const int timeout_ms = 2 * 1000;  /* Default 2 sec timeout for caller to send property. */
        int nr;
    ​
        if ((s = accept(property_set_fd, (struct sockaddr *) &addr, &addr_size)) < 0) {
            return;
        }
    ​
        /* Check socket options here */
        if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {
            close(s);
            ERROR("Unable to receive socket options\n");
            return;
        }
    ​
        ufds[0].fd = s;
        ufds[0].events = POLLIN;
        ufds[0].revents = 0;
        nr = TEMP_FAILURE_RETRY(poll(ufds, 1, timeout_ms));
        if (nr == 0) {
            ERROR("sys_prop: timeout waiting for uid=%d to send property message.\n", cr.uid);
            close(s);
            return;
        } else if (nr < 0) {
            ERROR("sys_prop: error waiting for uid=%d to send property message: %s\n", cr.uid, strerror(errno));
            close(s);
            return;
        }
    ​
        r = TEMP_FAILURE_RETRY(recv(s, &msg, sizeof(msg), MSG_DONTWAIT));
        if(r != sizeof(prop_msg)) {
            ERROR("sys_prop: mis-match msg size received: %d expected: %zu: %s\n",
                  r, sizeof(prop_msg), strerror(errno));
            close(s);
            return;
        }
    ​
        switch(msg.cmd) {
        case PROP_MSG_SETPROP:
            msg.name[PROP_NAME_MAX-1] = 0;
            msg.value[PROP_VALUE_MAX-1] = 0;
    ​
            if (!is_legal_property_name(msg.name, strlen(msg.name))) {
                ERROR("sys_prop: illegal property name. Got: \"%s\"\n", msg.name);
                close(s);
                return;
            }
    ​
            getpeercon(s, &source_ctx);
    ​
            if(memcmp(msg.name,"ctl.",4) == 0) {
                // Keep the old close-socket-early behavior when handling
                // ctl.* properties.
                close(s);
                if (check_control_mac_perms(msg.value, source_ctx, &cr)) {
                    handle_control_message((char*) msg.name + 4, (char*) msg.value);
                } else {
                    ERROR("sys_prop: Unable to %s service ctl [%s] uid:%d gid:%d pid:%d\n",
                            msg.name + 4, msg.value, cr.uid, cr.gid, cr.pid);
                }
            } else {
                if (check_mac_perms(msg.name, source_ctx, &cr)) {
                    property_set((char*) msg.name, (char*) msg.value);
                } else {
                    ERROR("sys_prop: permission denied uid:%d  name:%s\n",
                          cr.uid, msg.name);
                }
    ​
                // Note: bionic's property client code assumes that the
                // property server will not close the socket until *AFTER*
                // the property is written to memory.
                close(s);
            }
            freecon(source_ctx);
            break;
    ​
        default:
            close(s);
            break;
        }
    }
    ```

    handle\_property\_set\_fd()接收子进程请求并设置相关的属性值。
