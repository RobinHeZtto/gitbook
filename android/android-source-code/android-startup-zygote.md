# Android启动之Zygote

> Zygote是Android系统中的进程孵化器，它孵化了Android系统核心进程system\_server及所有的应用程序进程。此篇博客主要分析Zygote的启动流程。相关源码在以下文件中：
>
> &#x20;system/core/init/service.cpp frameworks/base/cmds/app\_process/app\_main.cpp frameworks/base/core/jni/AndroidRuntime.cpp frameworks/base/core/java/com/android/internal/os/ZygoteInit.java

![](<../../.gitbook/assets/image (385).png>)

Zygote以服务的形式定义在init.${ro.zygote}.rc文件中，并通过`import /init.${ro.zygote}.rc`的形式包含到/init.rc文件，其中ro.zygote是用来区分32与64位版本的属性值，在default.prop中定义ro.zygote属性值，其可取的值有"zygote32"，"zygote32\_64"，"zygote64"，"zygote64\_32"。下面是init.zygote64\_32.rc中的定义:

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
​
//根据以上的定义对应生成二个zygote进程
root:/ $ ps | grep zygote
root      754   1     2237660 82604 poll_sched 0000000000 S zygote64
root      755   1     1668956 69200 poll_sched 0000000000 S zygote
```

init.zygote64\_32.rc文件中定义了zygote与zygote\_secondary二个service，系统启动后可以看到存在zygote64与zygote二个进程，即分别通过/system/bin/下的app\_process64与app\_process32启动。`class main`定义zygote服务属于class main，通过`class_start main`启动，并为其创建名为"zygote"的socket用于进程间通信，如果zygote重启，audioserver，cameraserver，media，netd进程也将重新启动。 下面首先从init解析rc启动Service开始看Zygote的启动流程：

```
>>> system/core/init/service.cpp
bool Service::Start() {
    ......
    // fork service进程
    pid_t pid = fork();
    if (pid == 0) {
        // 创建/dev/socket/zygote
        for (const auto& si : sockets_) {
            int socket_type = ((si.type == "stream" ? SOCK_STREAM :
                                (si.type == "dgram" ? SOCK_DGRAM :
                                 SOCK_SEQPACKET)));
            const char* socketcon =
                !si.socketcon.empty() ? si.socketcon.c_str() : scon.c_str();
​
            int s = create_socket(si.name.c_str(), socket_type, si.perm,
                                  si.uid, si.gid, socketcon);
            if (s >= 0) {
                // 把socket的fd保存在环境变量ANDROID_SOCKET_zygote中
                PublishSocket(si.name, s);
            }
        }
​
        ......
​
        setpgid(0, getpid());
​
        // As requested, set our gid, supplemental gids, and uid.
        if (gid_) {
            if (setgid(gid_) != 0) {
                ERROR("setgid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (!supp_gids_.empty()) {
            if (setgroups(supp_gids_.size(), &supp_gids_[0]) != 0) {
                ERROR("setgroups failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (uid_) {
            if (setuid(uid_) != 0) {
                ERROR("setuid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (!seclabel_.empty()) {
            if (setexeccon(seclabel_.c_str()) < 0) {
                ERROR("cannot setexeccon('%s'): %s\n",
                      seclabel_.c_str(), strerror(errno));
                _exit(127);
            }
        }
​
        std::vector<char*> strs;
        for (const auto& s : args_) {
            strs.push_back(const_cast<char*>(s.c_str()));
        }
        strs.push_back(nullptr);
        // 执行app_process启动zygote
        if (execve(args_[0].c_str(), (char**) &strs[0], (char**) ENV) < 0) {
            ERROR("cannot execve('%s'): %s\n", args_[0].c_str(), strerror(errno));
        }
​
        _exit(127);
    }
​
    ......
    return true;
}
```

## app\_process

app\_process64/app\_process32是system/bin下的可执行文件，其源码在frameworks/base/cmds/app\_process/下，app\_process不仅可以用来启动Zygote进程还可以用来执行系统中的某个类(/system/bin/am就是通过app\_process来实现的)。它的的Usage如下：

> app\_process \[java-options] cmd-dir start-class-name \[options] java-options: 以"-"开头，启动虚拟机时传递给虚拟机的参数。 cmd-dir: cmd的目录，/system/bin。 start-class-name: 需要启动的java类，需包含静态main()方法。 options: "--zygote"表示启动zygote进程，"--application"表示启动应用程序进程。

下面从app\_process的main函数开始分析，main()中主要是对参数解析。

```
>>> frameworks/base/cmds/app_process/app_main.cpp
​
//AppRuntime继承自AndroidRuntime
class AppRuntime : public AndroidRuntime
{
public:
    AppRuntime(char* argBlockStart, const size_t argBlockLength)
        : AndroidRuntime(argBlockStart, argBlockLength)
        , mClass(NULL)
    {
    }
    ......
    String8 mClassName;
    Vector<String8> mArgs;
    jclass mClass;
};
​
​
int main(int argc, char* const argv[])
{
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        // Older kernels don't understand PR_SET_NO_NEW_PRIVS and return
        // EINVAL. Don't die on such kernels.
        if (errno != EINVAL) {
            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
            return 12;
        }
    }
​
    // 创建AppRuntime并保存参数，AppRuntime继承自AndroidRuntime，主要工作是创建加载ART虚拟机
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
​
    // 过滤掉argv[0]
    argc--;
    argv++;
​
    // runtime参数解析
    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }
​
    // --zygote : 启动zygote.
    // --start-system-server : 启动systemServer.
    // --application : 启动application.
    // --nice-name : 进程名
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;
​
    ++i;  // 跳过没有使用的参数"parent dir"，即/system/bin
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;　// zygote的进程名为zygote64或zygote
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
​
    // 准备执行参数
    Vector<String8> args;
    if (!className.isEmpty()) {
        // 指定执行class,非zygote模式
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // zygote模式，创建/data/dalvik-cache/arm64,/data/dalvik-cache/arm*缓存目录
        maybeCreateDalvikCache();
​
        // 添加zygote参数，start-system-server
        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }
​
        // 添加zygote参数,--abi-list
        // [ro.product.cpu.abilist64]: [arm64-v8a]
        // [ro.product.cpu.abilist32]: [armeabi-v7a,armeabi]
        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
​
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);
​
        // 所有参数将传递给ZygoteInit.main()
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
​
    // 设置进程名niceName
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }
​
    if (zygote) {
        // zygote模式执行ZygoteInit类
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        //　非zygote模式执行RuntimeInit类
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

app\_main的main函数主要是对参数进行解析，并判断启动模式zygote或者application,而后进入到AndroidRuntime执行。

## AndroidRuntime

接下来进入到AndroidRuntime::start中执行，主要工作是启动虚拟机并加载执行参数className指定的类。

```
>>>　frameworks/base/core/jni/AndroidRuntime.cpp
​
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // 启动时可以看到log信息 -> “AndroidRuntime: >>>>>> START com.android.internal.os.ZygoteInit uid 0 <<<<<<”
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());
​
    static const String8 startSystemServer("start-system-server");
​
    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }
​
    // 设置系统目录环境变量ANDROID_ROOT=/system
    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }
​
    // 使用JniInvocation启动虚拟机
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    // 调用AppRuntime的重载函数onVmCreated，zygote模式下直接返回
    onVmCreated(env);
​
    // 注册系统jni函数，将全局数组gRegJNI中的jni函数逐一注册
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
​
    // 构建执行类main()的String argv[]参数
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
​
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
​
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
​
    //将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
          　 // 执行com.android.internal.os.ZygoteInit或
            //　com.android.internal.os.RuntimeInit中的main()方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
            ......
        }
    }
    free(slashClassName);
}
```

AndroidRuntime::start中，先通过JNI invocation API创建加载虚拟机，然后加载执行ZygoteInit.java类main方法(启动虚拟机后即默认主线程)。创建加载虚拟机执行的是startVm()，下面是startVm()的具体实现：

```
>>>　frameworks/base/core/jni/AndroidRuntime.cpp
​
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
    JavaVMInitArgs initArgs;
​
    // 主要是通过addOption添加虚拟机启动参数,大多参数都是通过parseRuntimeOption从系统属性值中读取
    ......
​
    /* 以下是通过logcat打印出来的虚拟机启动参数
    01-06 13:44:08.556   757   757 I art     : option[0]=-Xzygote
    01-06 13:44:08.557   757   757 I art     : option[1]=-Xstacktracefile:/data/anr/traces.txt
    01-06 13:44:08.557   757   757 I art     : option[2]=exit
    01-06 13:44:08.558   757   757 I art     : option[3]=vfprintf
    01-06 13:44:08.558   757   757 I art     : option[4]=sensitiveThread
    01-06 13:44:08.559   757   757 I art     : option[5]=-verbose:gc
    01-06 13:44:08.560   757   757 I art     : option[6]=-Xms8m
    01-06 13:44:08.560   757   757 I art     : option[7]=-Xmx512m
    01-06 13:44:08.560   757   757 I art     : option[8]=-XX:HeapGrowthLimit=256m
    01-06 13:44:08.560   757   757 I art     : option[9]=-XX:HeapMinFree=512k
    01-06 13:44:08.561   757   757 I art     : option[10]=-XX:HeapMaxFree=8m
    01-06 13:44:08.562   757   757 I art     : option[11]=-XX:HeapTargetUtilization=0.75
    01-06 13:44:08.562   757   757 I art     : option[12]=-Xusejit:true
    01-06 13:44:08.562   757   757 I art     : option[13]=-Xjitsaveprofilinginfo
    01-06 13:44:08.563   757   757 I art     : option[14]=-agentlib:jdwp=transport=dt_android_adb,suspend=n,server=y
    01-06 13:44:08.563   757   757 I art     : option[15]=-Xlockprofthreshold:500
    01-06 13:44:08.563   757   757 I art     : option[16]=-Ximage-compiler-option
    01-06 13:44:08.570   757   757 I art     : option[17]=--runtime-arg
    01-06 13:44:08.571   757   757 I art     : option[18]=-Ximage-compiler-option
    01-06 13:44:08.571   757   757 I art     : option[19]=-Xms64m
    01-06 13:44:08.571   757   757 I art     : option[20]=-Ximage-compiler-option
    01-06 13:44:08.571   757   757 I art     : option[21]=--runtime-arg
    01-06 13:44:08.571   757   757 I art     : option[22]=-Ximage-compiler-option
    01-06 13:44:08.571   757   757 I art     : option[23]=-Xmx64m
    01-06 13:44:08.592   757   757 I art     : option[24]=-Ximage-compiler-option
    01-06 13:44:08.592   757   757 I art     : option[25]=--image-classes=/system/etc/preloaded-classes
    01-06 13:44:08.592   757   757 I art     : option[26]=-Ximage-compiler-option
    01-06 13:44:08.592   757   757 I art     : option[27]=--compiled-classes=/system/etc/compiled-classes
    01-06 13:44:08.593   757   757 I art     : option[28]=-Xcompiler-option
    01-06 13:44:08.593   757   757 I art     : option[29]=--runtime-arg
    01-06 13:44:08.593   757   757 I art     : option[30]=-Xcompiler-option
    01-06 13:44:08.593   757   757 I art     : option[31]=-Xms64m
    01-06 13:44:08.593   757   757 I art     : option[32]=-Xcompiler-option
    01-06 13:44:08.593   757   757 I art     : option[33]=--runtime-arg
    01-06 13:44:08.594   757   757 I art     : option[34]=-Xcompiler-option
    01-06 13:44:08.594   757   757 I art     : option[35]=-Xmx512m
    01-06 13:44:08.594   757   757 I art     : option[36]=-Ximage-compiler-option
    01-06 13:44:08.594   757   757 I art     : option[37]=--instruction-set-variant=kryo
    01-06 13:44:08.594   757   757 I art     : option[38]=-Xcompiler-option
    01-06 13:44:08.594   757   757 I art     : option[39]=--instruction-set-variant=kryo
    01-06 13:44:08.595   757   757 I art     : option[40]=-Ximage-compiler-option
    01-06 13:44:08.595   757   757 I art     : option[41]=--instruction-set-features=default
    01-06 13:44:08.595   757   757 I art     : option[42]=-Xcompiler-option
    01-06 13:44:08.595   757   757 I art     : option[43]=--instruction-set-features=default
    01-06 13:44:08.595   757   757 I art     : option[44]=-Duser.locale=zh-cn
    01-06 13:44:08.595   757   757 I art     : option[45]=--cpu-abilist=arm64-v8a
    01-06 13:44:08.596   757   757 I art     : option[46]=-Xfingerprint:TCL/london/london:7.0/London-V1.0.1.1C-11170539/jenkin11170539:userdebug/release-keys
    */
​
    initArgs.version = JNI_VERSION_1_4;
    initArgs.options = mOptions.editArray();
    initArgs.nOptions = mOptions.size();
    initArgs.ignoreUnrecognized = JNI_FALSE;
​
​
    // 通过JNI_CreateJavaVM创建加载虚拟机，其中JavaVM*整个进程只有一个，JNIEnv*每个线程一个,pEnv是保存jni接口的指针
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
​
    return 0;
}
```

## ZygoteInit

加载虚拟机后将调用到ZygoteInit的静态main方法对Zygote java层的初始化，下面继续分析ZygoteInit.main()的执行流程。

```
>>> frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
​
public static void main(String argv[]) {
    ......
    try {
        ......
        // 解析参数，根据参数决定是否启动SystemServer，获取abiList及socketName
        boolean startSystemServer = false;
        String socketName = "zygote";
        String abiList = null;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                socketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }
​
        if (abiList == null) {
            throw new RuntimeException("No ABI list supplied.");
        }
        // 注册zygote socket监听端口
        registerZygoteSocket(socketName);
​
        // 加载系统资源。zygote fork的子进程将继承zygote的虚拟机和加载的资源，以加快应用启动的速度。
        preload();
        ......
        // 启动SystemServer进程
        if (startSystemServer) {
            startSystemServer(abiList, socketName);
        }
​
        // 循环监听socket消息
        Log.i(TAG, "Accepting command socket connections");
        runSelectLoop(abiList);
​
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```

ZygoteInit.main()主要工作是预加载系统类与资源，启动systemServer，注册并监听socket连接消息。其中注册并监听socket连接消息内容在 **Android之Zygote-应用进程创建 分析，启动systemServer流程在 Android启动之SystemServer(上) 中分析，下面分析preload预加载系统类与资源的过程。**

![](<../../.gitbook/assets/image (87).png>)

**Linux中的进程通过系统调用fork产生后，其父子进程的内存映像(代码段，数据段，堆/栈)是共享的，只有子进程改写这些区域时才为子进程分配新的page(Copy On Write机制)，另外zygote fork子进程后并没有调用exec，即未替换掉zygote进程的代码段，数据段，堆栈，这样zygote fork出的子进程就可以共享它预加载的资源及类库了。 在preload中，zygote预加载系统常用类库及资源，以减短应用启动的时间。**

```
>>> frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
​
static void preload() {
    ......
    // 根据frameworks/base/preloaded-classes加载java类
    preloadClasses();
    ......
    // 根据frameworks/base/core/res/res/values/array.xml中
    // 的preloaded_drawables及preloaded_color_state_lists标签加载drawable及color资源
    preloadResources();
    ......
    preloadOpenGL();
    ......
    // 加载libandroid.so，libcompiler_rt.so，libjnigraphics.so
    preloadSharedLibraries();
    preloadTextResources();
    ......
    WebViewFactory.prepareWebViewInZygote();
    ......
}
```

**SystemServer进程的启动，及应用程序的创建在后面的博客中单独分析。**
