---
layout: single
related: false
title:  Android Systrace的部分Tag含义
date:   2021-03-09 21:52:00 +0800
categories: android
tags: display android graphics
toc: true
---

> systrace的一些tag标签的含义和作用。

<!-- more -->

# 1. CPU*（0-7）

Kernel内核模块，可以查看各个CPU执行了什么进程任务。

cpu信息的目录是`/sys/devices/system/cpu`，例如我的一加六老设备：

```s
OnePlus6:/sys/devices/system/cpu $ ls
core_ctl_isolated cpu4    cpuidle               isolated   possible 
cpu0              cpu5    gladiator_hang_detect kernel_max power    
cpu1              cpu6    hang_detect_gold      modalias   present  
cpu2              cpu7    hang_detect_silver    offline    uevent   
cpu3              cpufreq hotplug               online    
```

***

# 2. HW_VSYNC_ON_XXX

+ 值：1表示HW VSYNC信号被打开，0关闭
+ 出现时间：HW VSYNC硬件信号被打开/关闭的时候
+ 含义：

1. HW_VSYNC_ON_XXX后面的XXX表示display id（可通过dump SurfaceFlinger查看），用于区分不同屏幕的HW VSYNC硬件信号
2. HW VSYNC硬件信号之所以会有时候被打开/关闭，是因为目前Android Graphics/Display依赖的信号不是硬件信号，而是软件信号 —— DispSync。`硬件信号的主要作用是用于校准，打开的时机是当软件VSYNC的误差超过一定值后，DispSync会打开HW VSYNC硬件信号进行校准`。

+ 作用：用作分析硬件模块导致的BUG的时间点

*** 

## 2.1. fence同步机制简单释义

> BufferQueue中的Buffer在整个绘制、合成、显示的过程中，一直在 CPU，GPU 和 HWC 之前传递，某一方要使用 Buffer 之前，需要检查之前的使用者是否已经移交了 Buffer 的“使用权”。而这里的“使用权”，就是fence。当fence释放（即signal）的时候，说明 Buffer 的上一个使用者已经交出了使用权，对于 Buffer 进行操作是安全的。

在 Android 里面，总共有三类fence：`acquire fence，release fence 和 present fence`。其中acquire fence和release fence隶属于Layer，present fence隶属于帧（即 Layers）：

+ `acquire fence`：App将Buffer通过queueBuffer()还给BufferQueue的时候，此时该Buffer的GPU侧其实是还没有完成的，此时会带上一个fence，这个fence就是acquire fence。当SurfaceFlinger/HWC要读取Buffer以进行合成操作的时候，需要等acquire fence释放之后才行
+ `release fence`：当App通过dequeueBuffer()从BufferQueue申请Buffer，要对Buffer进行绘制的时候，需要保证HWC已经不再需要这个Buffer了，即需要等release fence signal才能对 Buffer进行写操作。
+ `present fence`：在HWC1的时候称为retire fence，在HWC2中改名为present fence。当前帧成功显示到屏幕的时候，`present fence就会signal`。

***

# 3. HW_VSYNC_XXX

+ 值：值发生变化（0->1 / 1->0）是表示当前时刻发出了HW VSYNC硬件信号
+ 出现时间：上面HW_VSYNC_ON_XXX打开的时候
+ 含义：

1. HW_VSYNC_XXX记录的是当前时刻收到了HW VYSNC硬件信号

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
//Step 1: 当收到硬件信号后，回调该函数
bool HWComposer::onVsync(hal::HWDisplayId hwcDisplayId, int64_t timestamp) {
    const auto displayId = toPhysicalDisplayId(hwcDisplayId);
    ......
    const auto tag = "HW_VSYNC_" + to_string(*displayId);
    ATRACE_INT(tag.c_str(), displayData.vsyncTraceToggle);
    //Step 2: 取反赋值给HW_VSYNC_XXX，即systrace中该值0和1之间的变化
    displayData.vsyncTraceToggle = !displayData.vsyncTraceToggle;

    return true;
}
```

***

# 4. hasClientComposition

+ 值：布尔值，1表示本次SurfaceFlinger合成存在GPU合成的layer，0表示本次合成的所有layer都不是GPU合成
+ 出现时间：SurfaceFlinger的doComposition合成开始
+ 含义：

1. Client合成（GPU合成）是通过CompositionEngine调用一系列的OpenGLES接口完成合成
2. Device合成（硬件合成）是使用Hardware Composer进行合成。

**优势**：省电

**劣势**：某些场景无法进行Device合成，比如Layer是圆角的，Layer的总数超过硬件上限

+ 作用：如果发现本应该是Device合成的场景出现hasClientComposition值为1的情况，则可结合dump SurfaceFlinger信息分析合成策略是否有问题

***

# 5. FrameMissed/GpuFrameMissed/HwcFrameMissed

+ 值：1表示`上一次合成`有FrameMissed/GpuFrameMissed/HwcFrameMissed，0则没有
+ 出现时间：SurfaceFlinger的INVALIDATE阶段开始
+ 含义：`FrameMissed/GpuFrameMissed/HwcFrameMissed表示的是上一次合成的结果，当SurfaceFlinger合成后显示到屏幕上显示一帧，present fence就会signal。因此可以将present fence signal作为一次合成完结的标志。`SurfaceFlinger每次开始被Vysnc-sf唤醒时，会先检查上一次合成情况，方式就是检查上一次合成的present fence有没有signal。如果没有，则认为是FrameMissed，并结合上一次合成方式是否有GPU或者HWC参与，同步GpuFrameMissed/HwcFrameMissed信息。

+ present fence没有及时signal主要有两种原因：

1. Display问题
2. APP/游戏的GPU负载过高：上层GPU负载过高会导致底层大部分时间都在等GPU渲染工作完成，延迟了present fence的signal，导致FrameMissed

+ FrameMissed作用：

1. 统计丢帧
2. Android有一个debug开关，可以在检测到上一帧有FrameMissed出现的时候，跳过本次的合成，留给底层更多的时间去显示。这个初衷是好的，不让底层过于繁忙，通过主动跳过合成来减缓底层的工作量。但是由于跳过合成就相当于主动丢帧，在某些场景下会导致到持续性的掉帧。因此这个开关一般是不会打开的。

***

# 6. VSYNC-sf/VSYNC-app

+ 值：表示SurfaceFlinger和APP发出Vsync信号（刷新率60是16.67ms，刷新率90是11.11ms）
+ 出现时间：DispSync分发的时候
+ 作用：分别针对SurfaceFlinger合成和APP应用渲染的起点。`如果一处没有Vsync-sf，则说明此处有丢帧情况，原因可能是上面的FrameMissed，或者是APP没有及时完成渲染导致丢帧`

## 6.1. VSYNC-app基本不变化的原因

现在绝大部分手游都使用游戏引擎，例如Unity、Unreal，这些引擎会自己去控制刷新率。

例如和平精英、王者荣耀可以有多个帧率档位选择，所以就不会通过Vsync-app的速率进行绘制刷新。

***

# 7. queueBuffer

Systrace中抓取的都是CPU侧的，例如每个CPU核的频率、C-State、运行了什么线程、线程间的调用关系、运行时长等。

而生产者通过dequeuebuffer获取buffer后，会最终交给GPU渲染绘制，然后通过queuebuffer将buffer还给BufferQueue。

`如果GPU渲染的时间长，则可以初步判断性能问题是出自于GPU侧`

## 7.1. FenceMonitor

Android Q在libgui库引入新的内部类FenceMonitor，作用是跟踪Fence的生命周期，在Systrace中展示一个Fence从产生到signal需要的时间。

1. **在Systrace中GPU Completion的每个`waiting for GPU completion ×××`的长度，大致可以作为GPU渲染所花费的时间（即acquire fence释放的总时间），但是并不严谨（解释如下）。**

通过这个时间，可以判断是否有GPU bound的现象。

2. **相对应的，`waiting for HWC release ×××`的长度大致可以作为release fence的释放总时间参考。在release fence signal之前，GPU是无法对dequeuebuffer拿到的Buffer进行读写的（因为此时Buffer还是归HWC所有）。**

通过这点，可以判断Display是否有问题。


**注**：systrace上的fence信息只能准确反映出signal的时间点，但无法反应出gpu/display开始干活的时间点。

再说原因：systrace上等待GPU fence的起始时间是从queueBuffer开始算的，而不是gpu真正开始干活时才算的。queueBuffer执行完成并不意味着gpu就能马上开始干活，有可能这个时候是因为display还没有释放（即signal）该buffer导致gpu不得不等在那里。所以我们不能说等待gpu的fence耗时就是gpu渲染的时间。同样的，HWC release fence也是一样的道理。

但有时候我们可以变相的计算出gpu的耗时：GPU complete signal的时间点，减去上一帧HWC release fence signal的时间点，这样计算出来的结果也只能是个近似值，因为从display buffer释放到gpu真正开始干活，这中间还有额外的准备时间。当然，有的时候systrace上反应不出HWC release fence signal的时间点，因为在dequeueBuffer的时候该release fence就已经释放了，所以上面的公式就排不上用场了。

***

FenceMonitor相关代码：

```cpp
//frameworks/native/libs/gui/Surface.cpp
//内部类
class FenceMonitor {
public:
    explicit FenceMonitor(const char* name) : mName(name), mFencesQueued(0), mFencesSignaled(0) {
        std::thread thread(&FenceMonitor::loop, this);
        pthread_setname_np(thread.native_handle(), mName);
        thread.detach();
    }
    ......
}

int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
    ATRACE_CALL();
    ALOGV("Surface::dequeueBuffer");
    .....
    if (CC_UNLIKELY(atrace_is_tag_enabled(ATRACE_TAG_GRAPHICS))) {
        //申请到Buffer
        static FenceMonitor hwcReleaseThread("HWC release");
        hwcReleaseThread.queueFence(fence);
    }
    .....
}

int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
    ATRACE_CALL();
    ALOGV("Surface::queueBuffer");
    .....
    if (CC_UNLIKELY(atrace_is_tag_enabled(ATRACE_TAG_GRAPHICS))) {
        //GPU绘制完成
        static FenceMonitor gpuCompletionThread("GPU completion");
        gpuCompletionThread.queueFence(fence);
    }

    return err;
}
```

# 8. 参考

+ [Android Systrace如何抓取分析问题](https://wizzie.top/Blog/2020/02/22/2020/200222_android_systrace_study/)
+ [Systrace 中的这些 tag 究竟是什么意思（一）](https://mp.weixin.qq.com/s/xgnXjhjPpJo27bbcDPW5RQ)
+ [如何通过 Systrace 查看 GPU 渲染花费的时间](https://mp.weixin.qq.com/s/dFAVVXUu1FkY7taZbNKCjQ)