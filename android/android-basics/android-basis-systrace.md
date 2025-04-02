# Android工具之Systrace与Perfetto使用

## 1. Systrace

Systrace 是 Android4.1 中新增的性能数据采样和分析工具。它可帮助开发者收集 Android 关键子系统（如 SurfaceFlinger/SystemServer/Kernel/Input/Display 等 Framework 部分关键模块、服务，View系统等）的运行信息，从而帮助开发者更直观的分析系统瓶颈，改进性能。

Systrace 的功能包括跟踪系统的 I/O 操作、内核工作队列、CPU 负载以及 Android 各个子系统的运行状况等。在 Android 平台中，它主要由3部分组成：

* **内核部分**：Systrace 利用了 Linux Kernel 中的 ftrace 功能。所以，如果要使用 Systrace 的话，必须开启 kernel 中和 ftrace 相关的模块。
* **数据采集部分**：Android 定义了一个 Trace 类。应用程序可利用该类把统计信息输出给ftrace。同时，Android 还有一个 atrace 程序，它可以从 ftrace 中读取统计信息然后交给数据分析工具来处理。
* **数据分析工具**：Android 提供一个 systrace.py（ python 脚本文件，位于 Android SDK目录/platform-tools/systrace 中，其内部将调用 atrace 程序）用来配置数据采集的方式（如采集数据的标签、输出文件名等）和收集 ftrace 统计数据并生成一个结果网页文件供用户查看。 从本质上说，Systrace 是对 Linux Kernel中 ftrace 的封装。应用进程需要利用 Android 提供的 Trace 类来使用 Systrace.\
  关于 Systrace 的官方介绍和使用可以看这里：

#### **1.1 使用方法:**

一般使用命令行的方式抓取systrace(Systrace工具在 Android-SDK 目录下的 platform-tools 里面), 然后使用chrome打开分析, 操作流程如下:

* 准备需要执行/抓取的界面
* 执行脚本并开始操作
* 设定好的时间到了之后，会将生成 Trace.html 文件，使用 **Chrome** 将这个文件打开进行分析

```
$ systrace.py -b 32768 -t 5 -o mytrace.html gfx input view webview wm am sm audio video camera hal app res dalvik rs bionic power sched irq freq idle disk mmc load sync workq memreclaim regulators
```

为了方便操作, 可在\~/.bashrc下配置命令的alias

```
alias trace='systrace.py gfx input view sched freq wm am hwui workq res dalvik sync disk load perf hal rs idle mmc'
```

后续可直接使用alias设置的命令抓取trace即可

```
trace -o mytrace.html
```

#### **1.2 命令参数:**

options可取值：

| options                                        | 解释                                |
| ---------------------------------------------- | --------------------------------- |
| -o `<FILE`>                                    | 输出的目标文件                           |
| -t N, –time=N                                  | 执行时间，默认5s                         |
| -b N, –buf-size=N                              | buffer大小（单位kB),用于限制trace总大小，默认无上限 |
| -k `<KFUNCS`>，–ktrace=`<KFUNCS`>               | 追踪kernel函数，用逗号分隔                  |
| -a `<APP_NAME`>,–app=`<APP_NAME`>              | 追踪应用包名，用逗号分隔                      |
| –from-file=`<FROM_FILE`>                       | 从文件中创建互动的systrace                 |
| -e `<DEVICE_SERIAL`>,–serial=`<DEVICE_SERIAL`> | 指定设备                              |
| -l, –list-categories                           | 列举可用的tags                         |

category可取值：

| category   | 解释                             |
| ---------- | ------------------------------ |
| gfx        | Graphics                       |
| input      | Input                          |
| view       | View System                    |
| webview    | WebView                        |
| wm         | Window Manager                 |
| am         | Activity Manager               |
| sm         | Sync Manager                   |
| audio      | Audio                          |
| video      | Video                          |
| camera     | Camera                         |
| hal        | Hardware Modules               |
| app        | Application                    |
| res        | Resource Loading               |
| dalvik     | Dalvik VM                      |
| rs         | RenderScript                   |
| bionic     | Bionic C Library               |
| power      | Power Management               |
| sched      | CPU Scheduling                 |
| irq        | IRQ Events                     |
| freq       | CPU Frequency                  |
| idle       | CPU Idle                       |
| disk       | Disk I/O                       |
| mmc        | eMMC commands                  |
| load       | CPU Load                       |
| sync       | Synchronization                |
| workq      | Kernel Workqueues              |
| memreclaim | Kernel Memory Reclaim          |
| regulators | Voltage and Current Regulators |

#### 1.3 快捷操作

导航操作:

| 导航操作 | 作用               |
| ---- | ---------------- |
| w    | 放大，\[+shift]速度更快 |
| s    | 缩小，\[+shift]速度更快 |
| a    | 左移，\[+shift]速度更快 |
| d    | 右移，\[+shift]速度更快 |

快捷操作:

| 常用操作 | 作用                          |
| ---- | --------------------------- |
| f    | **放大**当前选定区域                |
| m    | **标记**当前选定区域                |
| v    | 高亮**VSync**                 |
| g    | 切换是否显示**60hz**的网格线          |
| 0    | 恢复trace到**初始态**，这里是数字0而非字母o |

| 一般操作  | 作用                  |
| ----- | ------------------- |
| h     | 切换是否显示详情            |
| /     | 搜索关键字               |
| enter | 显示搜索结果，可通过← →定位搜索结果 |
| \`    | 显示/隐藏脚本控制台          |
| ?     | 显示帮助功能              |

对于脚本控制台，除了能当做记事本的功能，目前还不清楚有啥功能，或许还在开发中。

模式切换:

1. Select mode: **双击已选定区**能将所有相同的块高亮选中；（对应数字1）
2. Pan mode: 拖动平移视图（对应数字2）
3. Zoom mode:通过上/下拖动鼠标来实现放大/缩小功能；（对应数字3）
4. Timing mode:拖动来创建或移除时间窗口线。（对应数字4）

可通过按数字1\~4，用于切换鼠标模式； 另外，按住alt键，再滚动鼠标滚轮能实现放大/缩小功能。

## 2. Perfetto

Perfetto 是 Android 10 中引入的全新平台级跟踪工具。这是适用于 Android、Linux 和 Chrome 的更加通用和复杂的开源跟踪项目。与 Systrace 不同，它提供数据源超集，可让您以 protobuf 编码的二进制流形式记录任意长度的跟踪记录。可以在 [Perfetto 界面](https://ui.perfetto.dev/#!/)中打开这些跟踪记录。

Perfetto使用具体可参考:

[https://perfetto.dev/docs/quickstart/android-tracing](https://perfetto.dev/docs/quickstart/android-tracing)

## 3. 参考

* [https://developer.android.com/topic/performance/tracing/command-line](https://developer.android.com/topic/performance/tracing/command-line)
* [https://developer.android.com/topic/performance/tracing](https://developer.android.com/topic/performance/tracing)
