# Android启动之SystemServer(上)

> SystemServer是Android Framework的核心，大部分的Android系统核心服务都运行在SystemServer进程当中。此篇博客主要分析SystemServer的进程创建流程，下篇分析SystemServer的初始化流程，相关的源代码在以下文件中：
>
> * frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
> * frameworks/base/core/java/com/android/internal/os/Zygote.java
> * frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
> * frameworks/base/core/jni/com\_android\_internal\_os\_Zygote.cpp
> * frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
> * frameworks/base/core/jni/AndroidRuntime.cpp
> * frameworks/base/cmds/app\_process/app\_main.cpp

## 启动流程

SystemServer(system\_server进程)是通过Zygote fork生成的，如下init.zygote32.rc/init.zygote64\_32.rc中的定义，参数`--start-system-server`指定了Zygote启动SystemServer。

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks /sys/fs/cgroup/stune/foreground/tasks
​
service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
    class main
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
​
​
USER      PID   PPID  VSIZE  RSS   WCHAN              PC  NAME
root      1     0     16980  2636  SyS_epoll_ 0000500ab0 S /init
root      929   1     2166716 82876 poll_sched 7f8efff61c S zygote64
system    1695  929   2488940 152772 SyS_epoll_ 7f8efff4fc S system_server    
```

SystemServer的启动流程大概可以分为二个阶段，第一阶段Zygote fork system\_server进程，第二阶段执行SystemServer类，进行系统服务的启动及初始化，此篇博客分析SystemServer进程创建过程，下篇分析系统服务启动初始化过程。Zygote fork SystemServer的流程如下图示：

![](<../../.gitbook/assets/image (407).png>)

## ZygoteInit.main()

ZygoteInit.main()通过解析argv\[]参数列表来确定是否启动SystemServer，如果有定义参数`--start-system-server`则启动调用startSystemServer启动SystemServer

```
<<< frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
​
public static void main(String argv[]) {
    try {
        boolean startSystemServer = false;
        ......
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            }
            ......
        }
        ......
        if (startSystemServer) {
            startSystemServer(abiList, socketName);
        }
        ......
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```

## ZygoteInit.startSystemServer()

```
<<< frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
​
private static boolean startSystemServer(String abiList, String socketName)
        throws MethodAndArgsCaller, RuntimeException {
    long capabilities = posixCapabilitiesAsBits(
        OsConstants.CAP_IPC_LOCK,
        OsConstants.CAP_KILL,
        OsConstants.CAP_NET_ADMIN,
        OsConstants.CAP_NET_BIND_SERVICE,
        OsConstants.CAP_NET_BROADCAST,
        OsConstants.CAP_NET_RAW,
        OsConstants.CAP_SYS_MODULE,
        OsConstants.CAP_SYS_NICE,
        OsConstants.CAP_SYS_RESOURCE,
        OsConstants.CAP_SYS_TIME,
        OsConstants.CAP_SYS_TTY_CONFIG
    );
    /* Containers run without this capability, so avoid setting it in that case */
    if (!SystemProperties.getBoolean(PROPERTY_RUNNING_IN_CONTAINER, false)) {
        capabilities |= posixCapabilitiesAsBits(OsConstants.CAP_BLOCK_SUSPEND);
    }
    /* Hardcoded command line to start the system server */
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007,3009,3010",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system_server",
        "--runtime-args",
        "com.android.server.SystemServer",
    };
    ZygoteConnection.Arguments parsedArgs = null;
​
    int pid;
​
    try {
        // 准备执行参数
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
​
        // fork SystemServer并返回进程id
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }
​
    // 子进程即system_server进程中执行handleSystemServerProcess
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
​
        handleSystemServerProcess(parsedArgs);
    }
​
    return true;
}
```

startSystemServer中，首先为SystemServer准备启动参数，指定uid，gid为1000，进程名为“system\_server”，并指定执行类为“com.android.server.SystemServer”。然后执行Zygote.forkSystemServer() fork出进程，Zygote fork完成后将在子进程`system_server`中继续执行handleSystemServerProcess()。

## Zygote.forkSystemServer()

forkSystemServer()继续调用native方法nativeForkSystemServer()

```
<<< frameworks/base/core/java/com/android/internal/os/Zygote.java
​
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    VM_HOOKS.preFork();
    int pid = nativeForkSystemServer(
            uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
    // Enable tracing as soon as we enter the system_server.
    if (pid == 0) {
        Trace.setTracingEnabled(true);
    }
    VM_HOOKS.postForkCommon();
    return pid;
}
```

与fork普通应用进程实现一样，nativeForkSystemServer在frameworks/base/core/jni/com\_android\_internal\_os\_Zygote.cpp中实现，映射关联到com\_android\_internal\_os\_Zygote\_nativeForkSystemServer()

## nativeForkSystemServer()

```
<<< frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
​
{ "nativeForkSystemServer", "(II[II[[IJJ)I",
  (void *) com_android_internal_os_Zygote_nativeForkSystemServer },
​
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  // 与fork普通应用进程一样调用ForkAndSpecializeCommon创建出子进程
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL);
  if (pid > 0) {
      // 保存system_server进程pid
      gSystemServerPid = pid;
​
      int status;
      // 判断system_server是否启动成功，不成功则重启zygote
      if (waitpid(pid, &status, WNOHANG) == pid) {
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
      }
  }
  return pid;
}
```

与创建普通进程的执行流程一样，com\_android\_internal\_os\_Zygote\_nativeForkSystemServer()最终也是通过调用ForkAndSpecializeCommon()来创建system\_server进程，然后执行waitpid来判断system\_server是否退出(WNOHANG-若pid指定的子进程没有结束，则waitpid()函数返回0，不予以等待。若结束，则返回该子进程的ID)，如果返回了system\_server的pid说明system\_server启动失败，Zygote执行RuntimeAbort自杀，然后通过Init重启Zygote与system\_server。

```
<<< frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
​
// Utility routine to fork zygote and specialize the child process.
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jstring instructionSet, jstring dataDir) {                               
　SetSigChldHandler();　//设置SIGCHLD处理函数SigChldHandler
  ......                      
  pid_t pid = fork();
​
  if (pid == 0) {
    // The child process.
    ......
  } else if (pid > 0) {
    // the parent process
    ......
  }
  return pid;
}
```

ForkAndSpecializeCommon中通过设置SetSigChldHandler接收子进程退出的消息，如果接收到gSystemServerPid退出，那么Zygote将自杀，然后通过Init重启Zygote与system\_server。

```
<<< frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
​
static void SigChldHandler(int /*signal_number*/) {
  pid_t pid;
　......
  while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
    ......
    // If the just-crashed process is the system_server, bring down zygote
    // so that it is restarted by init and system server will be restarted
    // from there.
    if (pid == gSystemServerPid) {
      ALOGE("Exit zygote because system server (%d) has terminated", pid);
      kill(getpid(), SIGKILL);
    }
  }
  ......
}
```

在ForkAndSpecializeCommon中fork出子进程systemServer后，父子进程返回继续执行，子进程(pid == 0)继续执行handleSystemServerProcess。

## Zygote.handleSystemServerProcess()

在Zygote成功fork出子进程后，将在子进程system\_server中执行handleSystemServerProcess()

```
<<< frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
​
private static void handleSystemServerProcess(
        ZygoteConnection.Arguments parsedArgs)
        throws ZygoteInit.MethodAndArgsCaller {
    //关闭继承自父进程zygote的Socket
    closeServerSocket();
​
    //umask 0077后system_server创建的文件属性为0700
    Os.umask(S_IRWXG | S_IRWXO);
​
    //设置进程名"system_server",ps可以看到这个进程名
    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);　
    }
​
    // 获取环境变量SYSTEMSERVERCLASSPATH
    //SYSTEMSERVERCLASSPATH=/system/framework/services.jar:/system/framework/ethernet-service.jar:/system/framework/wifi-service.jar:/system/framework/container-service.jar
    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
        // 创建与installd的socket连接，对systemServerClasspath执行dex优化操作
        performSystemServerDexOpt(systemServerClasspath);
    }
​
    if (parsedArgs.invokeWith != null) {
      ......
    } else {
        ClassLoader cl = null;
        // 创建systemServer ClassLoader
        if (systemServerClasspath != null) {
            cl = createSystemServerClassLoader(systemServerClasspath,
                                               parsedArgs.targetSdkVersion);
​
            Thread.currentThread().setContextClassLoader(cl);
        }
​
        /*
         * Pass the remaining arguments to SystemServer.
         */
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
​
    /* should never reach here */
}
```

handleSystemServerProcess()中在关闭Zygote中复制而来的socket，设置进程名，并继续执行RuntimeInit.zygoteInit()。

## RuntimeInit.zygoteInit()

```
<<< frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
​
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
​
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams();
​
    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}
```

zygoteInit()中主要执行初始化相关的工作，最后将调到startClas(SystemServer)的main方法。commonInit中主要设置了默认的未捕获异常处理方法，timezone等。

```
<<< frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
​
private static final void commonInit() {
    // 设置Default未捕捉异常处理方法
    /* set default handler; this applies to all threads in the VM */
    Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
​
    /*
     * Install a TimezoneGetter subclass for ZoneInfo.db
     */
    TimezoneGetter.setInstance(new TimezoneGetter() {
        @Override
        public String getId() {
            return SystemProperties.get("persist.sys.timezone");
        }
    });
    TimeZone.setDefault(null);
​
    /*
     * Sets handler for java.util.logging to use Android log facilities.
     * The odd "new instance-and-then-throw-away" is a mirror of how
     * the "java.util.logging.config.class" system property works. We
     * can't use the system property here since the logger has almost
     * certainly already been initialized.
     */
    LogManager.getLogManager().reset();
    new AndroidConfig();
​
    /*
     * Sets the default HTTP User-Agent used by HttpURLConnection.
     */
    String userAgent = getDefaultUserAgent();
    System.setProperty("http.agent", userAgent);
​
    ......
​
    initialized = true;
}
```

nativeZygoteInit最终将调用到jni/AndroidRuntime.cpp中的onZygoteInit方法。

```
<<<　frameworks/base/core/jni/AndroidRuntime.cpp
​
{ "nativeZygoteInit", "()V",
    (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
​
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
​
virtual void onZygoteInit()
{
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();
}
```

onZygoteInit中创建SystemServer进程的ProcessState对象，打开并映射binder，并开启binder线程池。

```
>>> frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
​
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    ......
    设置虚拟机的内存利用率及targetSdkVersion
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
​
    final Arguments args;
    try {
        args = new Arguments(argv);
    } catch (IllegalArgumentException ex) {
        Slog.e(TAG, ex.getMessage());
        // let the process exit
        return;
    }
    ......
    // Remaining arguments are passed to the start class's static main
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
​
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl;
​
    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }
​
    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }
​
    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }
​
    /*
     * This throw gets caught in ZygoteInit.main(), which responds
     * by invoking the exception's run() method. This arrangement
     * clears up all the stack frames that were required in setting
     * up the process.
     */
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```

最终在invokeStaticMain中抛出MethodAndArgsCaller异常，并传递参数”com.android.server.SystemServer.main”,ZygoteInit.main()中catch该异常，并调用MethodAndArgsCaller.run反射执行SystemServer.main方法。

## MethodAndArgsCaller.run()

```
public void run() {
    try {
        mMethod.invoke(null, new Object[] { mArgs });
    } catch (IllegalAccessException ex) {
        throw new RuntimeException(ex);
    } catch (InvocationTargetException ex) {
        Throwable cause = ex.getCause();
        if (cause instanceof RuntimeException) {
            throw (RuntimeException) cause;
        } else if (cause instanceof Error) {
            throw (Error) cause;
        }
        throw new RuntimeException(ex);
    }
}
```
