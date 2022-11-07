---
layout: single
related: false
title:  Android ANR基本Log分析
date:   2020-08-06 21:52:00
categories: android
tags: graphics display android
toc: true
---

> ANR（Application Not Responding），字面意思是应用无响应，即用户的一些操作无法从应用中获取反馈。关于发生ANR的trace.txt文件的请参考[Android ANR traces.txt文件分析](https://wizzie.top/Blog/2020/06/11/2020/200611_android_tracetxt/)

<!--more-->

# 1. 触发原因

Android系统中的应用被Activity Manager及Window Manager两个系统服务监控着，Android系统会在如下情况触发ANR：

+ Input事件超过5s没有被处理完，即5秒内无法对输入事件（按键及触摸）做出响应
+ Service处理超时，前台20s，后台200s
+ BroadcastReceiver（广播接收器）处理超时，前台10S，后台60s
+ ContentProvider执行超时，比较少见

出现ANR之后一个直观现象就是系统会展示出一个ANR弹框。

从发生的原因分：

+ 主线程有耗时操作，如有复杂的layout布局，IO操作等。
+ 被Binder对端block
+ 被子线程同步锁block
+ Binder被占满导致主线程无法和SystemServer通信
+ 得不到系统资源（CPU/RAM/IO）

从进程的角度分：

+ 问题出在当前进程:
+ 主线程本身耗时, 或则主线程的消息队列存在耗时操作;
+ 主线程被本进程的其他子线程所blocked;
+ 问题出在远端进程(一般是binder call或socket等通信方式)

# 2. 基本log解读

```log
//进程是30970
//特殊情况下，如果PID是0，说明发生ANR之前，这个进程被LowMemoryKiller杀死了或者出现了Crash。这种情况下，是无法接收到系统的广播或者按键消息的，故而出现ANR
//ANR具体发生的包名
08-01 19:17:05.155  1000  1304  1328 I am_anr  : [0,30970,com.android.systemui,551042573,Input dispatching timed out (StatusBar, Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago.  Wait queue length: 54.  Wait queue head age: 9044.8ms.)]
//ANR具体发生的包名
08-01 19:17:28.258  1000  1304  1328 E ActivityManager: ANR in com.android.systemui
//ANR发生的原因是Input dispatching timed out
08-01 19:17:28.258  1000  1304  1328 E ActivityManager: PID: 30970
08-01 19:17:28.258  1000  1304  1328 E ActivityManager: Reason: Input dispatching timed out (StatusBar, Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago.  Wait queue length: 54.  Wait queue head age: 9044.8ms.)
//三个数字分别是1分钟、5分钟、15分钟内系统的平均负荷
//当CPU完全空闲的时候，平均负荷为0；当CPU工作量饱和的时候，平均负荷为1，通过Load可以判断系统负荷是否过重
//大致可以这样区分：
//当系统负荷持续大于0.7，你必须开始调查了，问题出在哪里，防止情况恶化。
//当系统负荷持续大于1.0，你必须动手寻找解决办法，把这个值降下来。
//当系统负荷达到5.0，就表明你的系统有很严重的问题
08-01 19:17:28.258  1000  1304  1328 E ActivityManager: Load: 46.53 / 37.82 / 34.77
//ANR发生的时候，Top进程的Cpu占用情况，user代表是用户空间，kernel是内核空间
//查看每个CPU的使用频度：adb shell cat /sys/devices/system/cpu/cpu1/cpufreq/stats/time_in_state
//一般如下规律：
//1. kswapd0 cpu占用率偏高，系统整体运行会缓慢，从而引起各种ANR。把问题转给"内存优化"，请他们进行优化
//2. logd　CPU占用率偏高，也会引起系统卡顿和ANR，因为各个进程输出LOG的操作被阻塞从而执行的极为缓慢
//3. Vold占用CPU过高，会引起系统卡顿和ANR，请负责存储的同学先调查
//4. qcom.sensor CPU占用率过高，会引起卡顿，请系统同学调查
//5. 应用自身CPU占用率较高，高概率应用自身问题
//6. 应用处于D状态，发生ANR，如果最后的操作是refriger，那么是应用被冻结了，正常情况下是功耗优化引起的。
08-01 19:17:28.258  1000  1304  1328 E ActivityManager: CPU usage from 0ms to 23321ms later (2020-08-01 19:17:04.850 to 2020-08-01 19:17:28.171) with 99% awake:
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   36% 30970/com.android.systemui: 25% user + 10% kernel / faults: 11945 minor 24 major
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   35% 20903/com.tencent.mobileqq:video: 28% user + 6.4% kernel / faults: 6413 minor 23 major
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   28% 555/surfaceflinger: 16% user + 12% kernel / faults: 1846 minor 2 major
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   26% 498/android.hardware.audio@5.0-service-***: 21% user + 4.9% kernel / faults: 13 minor
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   18% 1304/system_server: 9.5% user + 8.7% kernel / faults: 9666 minor 329 major
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   17% 3919/com.***.service: 10% user + 6.7% kernel / faults: 8604 minor 30 major
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   16% 28827/com.tencent.qqmusic: 9.2% user + 7.3% kernel / faults: 9299 minor 18 major
08-01 19:17:28.258  1000  1304  1328 E ActivityManager:   16% 142/kswapd0: 0% user + 16% kernel
...
```

***

# 3. 系统耗时分析方案

系统做一些耗时分析的操作会有一些Log标志：

1. `binder_sample`：

+ 功能说明: 监控每个进程的主线程的binder transaction的耗时情况, 当超过阈值时,则输出相应的目标调用信息，默认1000ms打开。
+ log格式: `52004 binder_sample (descriptor|3),(method_num|1|5),(time|1|3),(blocking_package|3),(sample_percent|1|6)`
+ log实例:

`2754 2754 I binder_sample: [android.app.IActivityManager,35,2900,android.process.media,5]`

从上面的log中可以得出:

+ 主线程2754;
+ 执行android.app.IActivityManager接口
+ 所对应方法code =35(即STOP_SERVICE_TRANSACTION),
+ 所花费时间为2900ms

该block所在package为 android.process.media，最后一个参数是sample比例(没有太大价值)

2. `dvm_lock_sample`

+ 功能说明: 当某个线程等待lock的时间blocked超过阈值,则输出当前的持锁状态 ;
+ log格式: `20003 dvm_lock_sample (process|3),(main|1|5),(thread|3),(time|1|3),(file|3),(line|1|5),(ownerfile|3),(ownerline|1|5),(sample_percent|1|6)`

`进程名，主线程？线程名，锁等待时间，下个持有者文件名，行号，上个持有者文件名（如果和下个相同，则是-），行号，等待百分比`

+ log实例:

`dvm_lock_sample: [system_server,1,Binder_9,1500,ActivityManagerService.java,6403,-,1448,0]`

意思是system_server: Binder_9,执行到ActivityManagerService.java的6403行代码,一直在等待AMS锁, 而该锁所同一文件的1448行代码所持有, 从而导致Binder_9线程被阻塞1500ms.

3. `binder starved`

+ 功能说明:当system_server等进程的线程池使用完, 无空闲线程时, 则binder通信都处于饥饿状态, 则饥饿状态超过一定阈值则输出信息;
+ 云控参数: persist.sys.binder.starvation  (默认值16ms)
+ log实例:

`1232 1232 "binder thread pool (16 threads) starved for 100 ms"`

解析: system_server进程的 线程池已满的持续长达100ms


***

# 4. kswapd0 CPU占用率很高

如果出现kswapd0 cpu 占用率很高，可以先查看内存使用情况。

## 4.1. /proc/meminfo内存使用信息

例如以下，可用内存只有62MB。

```log
------ MEMORY INFO (/proc/meminfo) ------
//所有可用RAM大小 （即物理内存减去一些预留位和内核的二进制代码大小）
//可以认为是系统可供分配的内存总大小, 通常大小会比实际物理内存小
MemTotal:        3844700 kB

//LowFree与HighFree的总和
//当前系统空闲的内存大小，对应所有处于NR_FREE_PAGES状态的页框
MemFree:           62100 kB

//MemFree + Active(file) + Inactive(file) + SReclaimable 此外还考虑了内存压力水位(watermark)的情况，计算比较复杂，详细见 si_mem_available(). 这只是理论上系统可用的内存，即理论上可回收的内存，但是实际上能用的达不到这么多
MemAvailable:     396584 kB

//用来给块设备做的缓冲大小（只记录文件系统的metadata以及 tracking in-flight pages，就是说 buffers是用来存储，目录里面有什么内容，权限等等。）
Buffers:           10160 kB
//用来给文件做缓冲大小（直接用来记忆我们打开的文件）. 它不包括SwapCached
Cached:           554704 kB
//已经被交换出来的内存，但仍然被存放在swapfile中。用来在需要的时候很快的被替换而不需要再次打开I/O端口
SwapCached:        25312 kB
//最近经常被使用的内存，除非非常必要否则不会被移作他用
Active:          1317228 kB
//最近不经常被使用的内存，非常用可能被用于其他途径
Inactive:         659868 kB
Active(anon):    1179772 kB
Inactive(anon):   403532 kB
Active(file):     137456 kB
Inactive(file):   256336 kB
Unevictable:      162512 kB
Mlocked:          162512 kB
//交换空间的总和
SwapTotal:       2113488 kB
//从RAM中被替换出暂时存在磁盘上的空间大小
SwapFree:         405628 kB
//等待被写回到磁盘的内存大小
Dirty:              1652 kB
//正在被写回到磁盘的内存大小
Writeback:            20 kB
AnonPages:       1571688 kB
//影射文件的大小
Mapped:           340044 kB
Shmem:              9360 kB
//内核数据结构缓存
Slab:             233948 kB
SReclaimable:      66952 kB
SUnreclaim:       166996 kB
KernelStack:       72480 kB
PageTables:       109660 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     4035836 kB
Committed_AS:   125397000 kB
//vmalloc内存大小
VmallocTotal:   263061440 kB
//已经被使用的虚拟内存大小
VmallocUsed:           0 kB
VmallocChunk:          0 kB
CmaTotal:         344064 kB
CmaFree:            2736 kB
...
```

## 4.2. dumpsys meminfo

直接dumpsys meminfo，是查看整体的内存占用情况，具体的还是需要加上process name。

```log
//对应上面MemTotal
Total RAM: 3,844,700K (status normal)

//cached pss对应变量cachedPss的值。这部分进程占用的内存并没有被释放，而由于他们都已切换到后台，且adj较低，系统认为可以释放掉这部分内存。所以对于这部分进程，系统最好有机制能及时清理掉从而释放内存。
//cached kernel对应"Buffers+Cached+SReclaimable-Mapped"这部分的内存由于理论上是可以被Kernel回收的，所以这里也计算在free中，但是这是一个理论上的值，实际上很难做到全部回收。
//free对应MemFree
 Free RAM: 1,003,024K (  400,796K cached pss +   509,892K cached kernel +     1,136K cached ion +    91,200K free)

//kernel对应"Shmem+SUnreclaim+VmallocUsed+PageTables+KernelStack",其中VmallocUsed是统计/proc/vmallocinfo中除ioremap,map_lowmem,vm_map_ram之外的和
//详细见"Debug.get_allocated_vmalloc_memory()"这部分即是对kernel的内存占用的一个统计，如果要统计kernel的内存占用，这个稍微准确一些
 Used RAM: 4,340,042K (3,862,614K used pss +   477,428K kernel)

//对应MemTotal - (totalPss - totalSwapPss) - MemFree - (cached kernel) - (kernel) - zramtotal
 Lost RAM:   135,889K

//第一个数 用变量zramtotal来代替，表示zram实际占用的物理内存，是从/sys/block/zram0/mm_stat中统计而来
//第二个数 对应 SwapTotal - SwapFree , 是已经在swap区的内存大小
//第三个数 对应 SwapTotal， 是整个swap区的大小
     ZRAM:   528,592K physical used for 1,757,284K in swap (2,113,488K total swap)
   Tuning: 384 (large 512), oom   645,120K, restore limit   107,520K (high-end-gfx)
```

# 5. 参考

+ [应用与系统稳定性第一篇---ANR问题分析的一般套路](https://www.imooc.com/article/280763)
+ [Android应用ANR分析](https://www.jianshu.com/p/30c1a5ad63a3)
+ [android 查看内存使用情况](https://blog.csdn.net/shift_wwx/article/details/42490863)
+ [Android内存占用分析](https://zhuanlan.zhihu.com/p/90076117)