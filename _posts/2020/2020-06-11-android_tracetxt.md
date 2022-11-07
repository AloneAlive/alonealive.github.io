---
layout: single
related: false
title:  Android ANR traces.txt文件分析
date:   2020-06-11 23:52:00
categories: android
tags: display
toc: true
---

> `trace.txt`生成:当APP(包括系统APP和用户APP)进程出现ANR、应用响应慢或WatchDog的监视没有得到回馈时,系统会dump此时的top进程,进程中Thread的运行状态就都dump到这个Trace文件中了。
> 
> ANR:Application Not Responding，即应用无响应

<!--more-->


# 1. ANR类型

一般有三种类型:

1. KeyDispatchTimeout(5 seconds) --主要类型：按键或触摸事件在特定时间内无响应
2. BroadcastTimeout(10 seconds)  --BroadcastReceiver：在特定时间内无法处理完成
3. ServiceTimeout(20 seconds) --小概率类型：Service在特定的时间内无法处理完成

另外还有`ProviderTimeout`和`WatchDog看门狗`等导致的ANR。

还有当系统内存或CPU资源不足时容易出现ANR， 一般这种情况会有`lowmemorykill`的log打印。

应用ANR产生的时候，在`ActivityManagerService`中会调用`appNotResponding`方法, 然后在`/data/anr/traces.txt`文件中写入ANR相关信息。

# 2. trace.txt获取

1. `adb shell`进入手机的`/data/anr`文件目录下面查看生成的`trace.txt`文件(如果`ls`查看文件列表没有权限,可以先`adb root`一下)
2. `adb pull /data/anr/` 将该文件导出,然后分析

log打印了ANR的基本信息(`adb shell top`查看进程, `adb logcat -v process |grep PID`查看日志), 可以分析`CPU使用率`得知ANR的简单情况;

如果CPU使用率很高,接近100%,可能是在进行大规模的计算更可能是陷入死循环;

如果CUP使用率很低,说明主线程被阻塞了,并且当IOwait很高,可能是主线程在等待I/O操作的完成。

对于ANR只是分析Log， 很难知道问题所在,我们还需要通过`Trace文件分析stack调用情况`,在log中显示的pid在traces文件中与之对应, 然后通过查看堆栈调用信息分析ANR的代码。

注:trace 文件的分析参考 https://blog.csdn.net/qq_25804863/article/details/49111005

***

# 3. Trace分析

Traces中显示的线程状态都是C代码定义的，可以通过查看线程状态对应的信息分析ANR问题。

如:

+ `TimedWaiting`对应的线程状态是TIMED_WAITING；
+ `kTimedWaiting, // TIMED_WAITING TS_WAIT in Object.wait() with a timeout`执行了无超时参数的wait函数；
+ `kSleeping, // TIMED_WAITING TS_SLEEPING in Thread.sleep()`执行了带有超时参数的 sleep 函数；
+ ZOMBIE                    线程死亡,终止运行
+ RUNNING/RUNNABLE          线程可运行或正在运行
+ TIMED_WAIT                执行了带有超时参数的 wait、sleep 或 join 函数
+ MONITOR                   线程阻塞,等待获取对象锁
+ WAIT                      执行了无超时参数的 wait 函数
+ INITIALIZING              新建,正在初始化,为其分配资源
+ STARTING                  新建,正在启动
+ NATIVE                    正在执行 JNI 本地函数
+ VMWAIT                    正在等待 VM 资源
+ SUSPENDED                 线程暂停,通常是由于 GC 或 debug 被暂停
