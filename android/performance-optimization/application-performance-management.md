---
description: 应用性能监控
---

# APM



![](<../../.gitbook/assets/image (379).png>)

## Crash:

监控：

Java Crash

* UncaughtExceptionHandler

Native Crash:

* [Android 平台 Native 代码的崩溃捕获机制及实现](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?)

## 卡顿：

**什么是卡顿？**

从人眼结构来说，当一组动作在 1 秒内有 12 次变化（即 12FPS），我们会认为这组动作是连贯的；而当大于 60FPS 时，人眼很难区分出来明显的变化，所以 60FPS 也一直作为业界衡量一个界面流畅程度的重要指标。一个稳定在 30FPS 的动画，我们不会认为是卡顿的，**但一旦 FPS 很不稳定，人眼往往容易感知到**。

**FPS 低并不意味着卡顿发生，而卡顿发生 FPS 一定不高。**

什么是**掉帧**（跳帧）？ 按照理想帧率 60FPS 这个指标，计算出平均每一帧的准备时间有 1000ms/60 = 16.6667ms，如果一帧的准备时间超出这个值，则认为发生掉帧，超出的时间越长，掉帧程度越严重。假设每帧准备时间约 32ms，每次只掉一帧，那么 1 秒内实际只刷新 30 帧，即平均帧率只有 30FPS，但这时往往不会觉得是卡顿。反而如果出现某次严重掉帧（>300ms），那么这一次的变化，通常很容易感知到。所以界面的掉帧程度，往往可以更直观的反映出卡顿。

**怎么衡量流程性**

我们将掉帧数划分出几个区间进行定级，掉帧数小于 3 帧的情况属于最佳，依次类推，见下表：

| Best   | Normal | Middle  | High     | Frozen  |
| ------ | ------ | ------- | -------- | ------- |
| \[0:3) | \[3:9) | \[9:24) | \[24:42) | \[42:∞) |

相比单看平均帧率，掉帧程度的分布可以明显的看出，界面卡顿（平均帧率低）的原因是因为连续轻微的掉帧（下图1），还是某次严重掉帧造成的（下图2）。  再通过 Activity 区分不同场景，计算每个界面在有效绘制的时间片内，掉帧程度的分布情况及平均帧率，从而来评估出一个界面的整体流畅程度。&#x20;

![Different Machine](https://github.com/Tencent/matrix/wiki/images/trace/diff_machine.png)

![Drop Frame](https://github.com/Tencent/matrix/wiki/images/trace/drop_frame.png)



1、 如何有效地监控到App发生卡顿，同时在发生卡顿时正确记录app的状态，如堆栈信息，CPU占用，内存占用，IO使用情况等等；

2、 统计到的卡顿信息上报到监控平台，需要处理分析分类上报内容，并通过平台Web直观简便地展示，供开发跟进处理。

一般主线程过多的UI绘制、大量的IO操作或是大量的计算操作占用CPU，导致App界面卡顿。只要我们能在发生卡顿的时候，捕捉到主线程的堆栈信息和系统的资源使用信息，即可准确分析卡顿发生在什么函数，资源占用情况如何。那么问题就是如何有效检测Android主线程的卡顿发生，目前业界两种主流有效的app监控方式如下：

1、 利用UI线程的Looper打印的日志匹配；

2、 使用Choreographer.FrameCallback





| <p><a href="https://github.com/Tencent/matrix"><br></a></p> | [Matrix-TraceCanary](https://github.com/Tencent/matrix) | TraceView | [BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor) | [BlockCanaryEx](https://github.com/seiginonakama/BlockCanaryEx) | [ArgusAPM](https://github.com/Qihoo360/ArgusAPM) | [ANR-WatchDog](https://github.com/SalomonBrys/ANR-WatchDog) | [广研卡顿系统](http://www.10tiao.com/html/330/201801/2653579565/1.html) |
| ----------------------------------------------------------- | ------------------------------------------------------- | --------- | -------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------ | ----------------------------------------------------------- | ----------------------------------------------------------------- |
| 定位精确度                                                       | 高                                                       | 高         | 中                                                                    | 高                                                               | 中                                                | 中                                                           | 中                                                                 |
| 字节码修改                                                       | 是                                                       | 否         | 否                                                                    | 是                                                               | 否                                                | 否                                                           | 否                                                                 |
| 资源占用                                                        | 较低                                                      | 较低        | 较低                                                                   | 较低                                                              | 较低                                               | 较低                                                          | 较低                                                                |
| 线上采集                                                        | 支持                                                      | 不支持       | 支持                                                                   | 支持                                                              | 支持                                               | 支持                                                          | 支持                                                                |
| 性能损耗                                                        | 可忽略                                                     | 非常大       | 可忽略                                                                  | 大                                                               | 可忽略                                              | 可忽略                                                         | 可忽略                                                               |
| 界面展示                                                        | 不支持                                                     | 支持        | 支持                                                                   | 支持                                                              | 支持                                               | 支持                                                          | 未知                                                                |
| 帧率计算                                                        | 支持                                                      | 不支持       | 不支持                                                                  | 不支持                                                             | 支持                                               | 不支持                                                         | 支持                                                                |
| ANR监控                                                       | 支持                                                      | 不支持       | 不支持                                                                  | 不支持                                                             | 支持                                               | 支持                                                          | 支持                                                                |
| 启动耗时                                                        | 支持                                                      | 不支持       | 不支持                                                                  | 不支持                                                             | 不支持                                              | 不支持                                                         | 不支持                                                               |





[https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary)

[https://cloud.tencent.com/developer/article/1611773](https://cloud.tencent.com/developer/article/1611773)[https://cloud.tencent.com/developer/article/1071811](https://cloud.tencent.com/developer/article/1071811)\
[https://cloud.tencent.com/developer/article/1156121](https://cloud.tencent.com/developer/article/1156121)\
[https://cloud.tencent.com/developer/article/1089920](https://cloud.tencent.com/developer/article/1089920)\
[https://cloud.tencent.com/developer/article/1373826](https://cloud.tencent.com/developer/article/1373826)
