---
layout: single
related: false
title:  Android SurfaceFlinger和HWC2概述
date:   2019-12-22 23:32:00
categories: android
tags: graphics
toc: true
---

> Android SurfaceFlinger和HWC2概述
> 参考Android Q AOSP源码添加修改部分内容
> 参考源码： http://aosp.opersys.com/xref/android-10.0.0_r14/

<!--more-->

# 1. SurfaceFlinger概述

**大多数APP在屏幕通常显示三个部分：**
+ 屏幕顶部的状态栏
+ 底部或者侧边的导航栏
+ 应用的界面

有些应用会显示更多或者更少的层。例如主屏幕会有一个单独的壁纸层；全屏幕的游戏可能会隐藏状态栏目。这些可以通过`Dump Surfacelinger`查看BufferLayers部分的信息来获取具体信息（`adb shell dumpsys SurfaceFlinger`）。从Dump结果看，layer呈树形结构(`Tree`)分布。

每个层都可以单独更新。状态栏和导航栏由系统进程渲染，而应用层由应用渲染，两者之间不进行协调。

***

## 1.1. SurfaceFlinger类定义

```cpp
//frameworks/native/services/surfaceflinger/SurfaceFlinger.h
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback
{
public:
    SurfaceFlingerBE& getBE() { return mBE; }
    const SurfaceFlingerBE& getBE() const { return mBE; }
    ......
```

继承BnSurfaceComposer，实现ISurfaceComposer接口；实现ComposerCallback；继承辅助类PriorityDumper，主要提供SurfaceFlinger的Dump信息，并且提高提供信息的分离和格式设置。

***

## 1.2. ISurfaceComposer接口实现

ISurfaceComposer是提供给上层Client端的接口（Bp端），此处的SurfaceFlinger是Server端（Bn端）。接口内容包括：

```cpp
//frameworks/native/include/gui/ISurfaceComposer.h
class ISurfaceComposer: public IInterface {
public:
    DECLARE_META_INTERFACE(SurfaceComposer)
    ......
        /* returns information for each configuration of the given display
     * intended to be used to get information about built-in displays */
    virtual status_t getDisplayConfigs(const sp<IBinder>& display,
            Vector<DisplayInfo>* configs) = 0;
    ......
```

接口在SurfaceFlinger中都有对应的方法实现。Client端通过Binder跨进程调到SurfaceFlinger中。获取Display的信息，其实现就是SurfaceFlinger的`getDisplayConfig`函数。

***

## 1.3. ComposerCallback接口实现

ComposerCallback是HWC2的callback接口，包括以下接口：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.h
namespace HWC2 {

class Display;
class Layer;

// Implement this interface to receive hardware composer events.
//
// These callback functions will generally be called on a hwbinder thread, but
// when first registering the callback the onHotplugReceived() function will
// immediately be called on the thread calling registerCallback().
//
// All calls receive a sequenceId, which will be the value that was supplied to
// HWC2::Device::registerCallback(). It's used to help differentiate callbacks
// from different hardware composer instances.
class ComposerCallback {
 public:
    virtual void onHotplugReceived(int32_t sequenceId, hwc2_display_t display,
                                   Connection connection) = 0;
    virtual void onRefreshReceived(int32_t sequenceId,
                                   hwc2_display_t display) = 0;
    virtual void onVsyncReceived(int32_t sequenceId, hwc2_display_t display,
                                 int64_t timestamp) = 0;
    virtual ~ComposerCallback() = default;
};
.....
```

Callback提供了注册接口`registerCallback`，在SurfaceFlinger初始化的时候注册：

```cpp
void SurfaceFlinger::init() {
    ALOGI(  "SurfaceFlinger's main thread ready to run. "
            "Initializing graphics H/W...");
....
mCompositionEngine->getHwComposer().registerCallback(this, getBE().mComposerSequenceId);
....
```

此处`registerCallback`的`this`就是SurfaceFlinger对ComposerCallback接口的实现。
+ onHotplugReceived： 热插拔事件的回调，显示屏幕连接或者断开时回调。
+ onRefreshReceived： 接收底层HWComposer的刷新请求。在`repaintEverythingForHWC`中，`mRepaintEverything`为`true`的时候，将触发一次刷新，重新进行合成显示。重新绘制说明底层配置、参数等有变动，SurfaceFlinger前面给的数据不能用，需要重新根据变动后的配置进行合成，给适合当前配置的显示数据。

```cpp
//frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onRefreshReceived(int sequenceId, hwc2_display_t /*hwcDisplayId*/) {
    Mutex::Autolock lock(mStateLock);
    if (sequenceId != getBE().mComposerSequenceId) {
        return;
    }
    repaintEverythingForHWC();
}

void SurfaceFlinger::repaintEverythingForHWC() {
    mRepaintEverything = true;
    mEventQueue->invalidate();
}
```

+ onVsyncReceived： Vsync事件上报，接收底层硬件上报的垂直同步信号。此处可以通过抓取`Systrace`的方式查看具体的Vsync的信息（底层硬件、SurfaceFlinger、APP三部分的Vsync，一般Android版本升级的时候会进行`vsync的tuning`）
+ **显示周期Vsync**： 设备显示会按照一定速率更新（一般是一秒60帧，即16.6ms刷新一次）。如果显示内容在刷新期间更新，则会出现撕裂现象，因此必须在周期之间更新（`这也是vsync tunning的必要性，保持SurfaceFlinger和draw frame都在vsync周期里面，并且不重叠`）在可以安全更新内容时，系统便会接收来自显示设备的信号。

刷新率可能会随时间而变化，例如一些设备的刷新范围在58fps至62fps之间，具体视当前条件而定。对于连接了HDMI的电视，刷新率在理论上可以下降到24Hz或者48Hz，以便和视频匹配。由于每个刷新周期只能更新屏幕一次，因此以200fps的刷新率为显示设备提交缓冲区并没有必要性，因为大部分桢不能被看到（人眼合适的是60fps）。SurfaceFlinger不会在应用提交缓冲区时进行操作，而是在显示设备准备好接收新缓冲区的时候才会唤醒。

当Vsync信号到达的时候，SurfaceFlinger会遍历层列表，以寻找新的缓冲区。如果找到会获取该缓冲区，否则会使用以前获取的缓冲区。SurfaceFlinger总是需要可显示的内容，因此会保留一个缓冲区。如果在某个层没有提交缓冲区，则该层会被忽略。

此处会在合成调用到`handlePageFlip`函数，函数中先调用`latchBuffer`从BufferQueue取Buffer，然后等待Vsync信号更新到FrameBuffer。

+ **合成方式**： 目前SurfaceFlinger支持两种合成方式：一种是Device合成，一种是Client合成。SurfaceFlinger在收集可见层的所有缓冲区之后，便会询问HardwareComposer应该如何进行合成。
+ + Client合成：之前称之为GLES合成，也可以称之为GPU合成，该合成方式是相对于硬件合成来说的，将各个Layer的内容用GPU渲染到暂存缓冲区中，最后将暂存缓冲区传送到显示硬件Client合成采用RenderEngine进行合成。
+ + Device合成： 用专门的硬件合成器进行合成HWComposer，所以硬件合成的能力就取决于硬件的实现。其合成方式是将各个Layer的数据全部传给显示硬件，并告知它从不同的缓冲区读取屏幕不同部分的数据。HWComposer是Device合成的抽象。

合成方式可以从Dump SurfaceFlinger中查看到Layer的具体合成方式，GPU合成一般可以通过开发者选项中启动，强制GPU合成；而Device合成在Dump信息中一般显示成SDE合成。

GPU合成数据后，作为一个特殊的Layer传给显示硬件。

```shell
Display 0 HWC layers:
-------------------------------------------------------------------------------
 Layer name
           Z |  Comp Type |   Disp Frame (LTRB) |          Source Crop (LTRB)
-------------------------------------------------------------------------------
 com.android.systemui.ImageWallpaper#0
  rel      0 |     Client |    0    0 1080 2280 |    0.0    0.0 1080.0 2280.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 net.oneplus.launcher/net.oneplus.launcher.Launcher#0
  rel      0 |     Client |    0    0 1080 2280 |    0.0    0.0 1080.0 2280.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 StatusBar#0
  rel      0 |     Client |    0    0 1080   80 |    0.0    0.0 1080.0   80.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 GestureButtonRegion#0
  rel      0 |     Client |    0 2216 1080 2280 |    0.0    0.0 1080.0   64.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 ScreenDecorOverlay#0
  rel      0 |     Device |    0    0 1080  106 |    0.0    0.0 1080.0  106.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 ScreenDecorOverlayBottom#0
  rel      0 |     Device |    0 2198 1080 2280 |    0.0    0.0 1080.0   82.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

|-----|------------|-----------|------|-------------|--------------------------|---------------------|---------------------|----|------------|-----------|----|-----|
| Idx |  Comp Type |   Split   | Pipe |    W x H    |          Format          |  Src Rect (L T R B) |  Dst Rect (L T R B) |  Z |    Flags   | Deci(HxV) | CS | Rng |
|-----|------------|-----------|------|-------------|--------------------------|---------------------|---------------------|----|------------|-----------|----|-----|
|   6 | GPU_TARGET |    Pipe-1 |   94 | 1088 x 2288 |           RGBA_8888_UBWC |    0    0 1080 2280 |    0    0 1080 2280 |  0 | 0x00000002 |   0 x   0 |  0 |   0 |
|   4 |        SDE |    Pipe-1 |  103 | 1088 x  112 |           RGBA_8888_UBWC |    0    0 1080  106 |    0    0 1080  106 |  1 | 0x00000000 |   0 x   0 |  1 |   1 |
|   5 |        SDE |    Pipe-1 |   92 | 1088 x   96 |           RGBA_8888_UBWC |    0    0 1080   82 |    0 2198 1080 2280 |  2 | 0x00000000 |   0 x   0 |  1 |   1 |
|-----|------------|-----------|------|-------------|--------------------------|---------------------|---------------------|----|------------|-----------|----|-----|
```
+ SurfaceFlingerBE: 从Android P上分离出来，定义上看是将Surfacelinger分离为前后端。
+ 消息队列和主线程： 和应用进程类似，SurfaceFlinger也有一个主线程，主要是进行显示数据的处理，即合成。Surfacelinger是一个服务，将会响应上层的请求，各个进程的请求都在SurfaceFlinger的各个Binder线程中，如果线程很耗时，那么应用端就会被block。主线程将他们分离开来，各干各的。

**Note：**
1. SurfaceFligner有两个状态，Layer也有两个状态，一个是mCurrentState，一个是mDrawingState。
2. 两个EventThread，一个是给SurfaceFlinger本身使用，一个是为了给应用分发事件的。

***

## 1.4. `mCurrentState`和`mDrawingState`

1.这两个成员是Layer类中`Layer::State`的类型。

```cpp
//Layer.h
    struct State {
        Geometry active;  //计算后的实际尺寸
        Geometry requested; //用户设置的尺寸
        int32_t z; //Layer的Z轴值，值越小位置就越靠小
        uint32_t layerStack;  //和显示设备的关联值

        unit8_t alpha; //Layer的透明度
        uint8_t flags;  //Layer的标志（如果上次绘制后用户改变了Layer）
        uint8_t reserved[2];
        int32_t sequence; //序列值，Layer的属性变化一次就会加一（例如setAlpha,setSize,setLayer等）
        ...
        // the transparentRegion hint is a bit special, it's latched only
        // when we receive a buffer -- this is because it's "content"
        // dependent.
        Region activeTransparentRegion; //实际的透明区域
        Region requestedTransparentRegion;  //用户中的透明区域
        ...
    };
```

2.Surfacelinger创建Surface的时候，会调用`createLayer`，然后调用`addClientLayer`函数，这里会把Layer对象放在`mCurrentState`的layerSortedByZ对象中。

3.Surfacelinger合成的时候，调用`preComposition`函数，会先调用`mDrawingState`的layerSortedByZ来获取上次绘图的Layer层列表（并不是所有layer都参与屏幕图像的绘制，因此通过State对象记录参与绘制的Layer对象）

4.Layer对象在绘制图形时，使用的是mDrawingState变量；用户调用接口设置Layer对象属性时，设置的值保存在mCurrentState中。这样就不会因为用户的操作而干扰Layer对象的绘制了。

5.`Layer::doTransaction`函数会比较这两个成员变量，如果有不同的地方，说明上次绘制后，用户改变了Layer的属性，要把这种变化通过flags返回。

6.`layerStack`字段是用户指定的一个值，用户可以给DisplayDevice指定一个layerStack值，只有Layer对象和DisplayDevice对象的layerStack相等，这个Layer才能在这个显示设备输出。这样的好处可以让显示设备只显示某个Surface的内容。例如，可以让HDMI显示设备只显示手机上播放的Surface窗口，但是不显示Activity窗口。

7.`Layer::doTransaction`最后会调用`commitTransaction`函数，就是将mCurrentState赋值给mDrawingState。

***

8.以上的是在Layer.cpp中的两个成员变量，而在`SurfaceFlinger.cpp`也有同名的`mCurrentState`和`mDrawingState`两个成员变量（定义在SurfaceFlinger.h中），定义不一样，只是名字相同。

```cpp
//SF.h
    class State {
    public:
        explicit State(LayerVector::StateSet set) : stateSet(set), layersSortedByZ(set) {}
        State& operator=(const State& other) {
            // We explicitly don't copy stateSet so that, e.g., mDrawingState
            // always uses the Drawing StateSet.
            layersSortedByZ = other.layersSortedByZ;
            displays = other.displays;
            colorMatrixChanged = other.colorMatrixChanged;
            if (colorMatrixChanged) {
                colorMatrix = other.colorMatrix;
            }
            return *this;
        }

        const LayerVector::StateSet stateSet = LayerVector::StateSet::Invalid;
        LayerVector layersSortedByZ;  //保存所有参与绘制的Layer对象
        DefaultKeyedVector< wp<IBinder>, DisplayDeviceState> displays; //保存所有输出设备的DisplayDeviceState对象

        bool colorMatrixChanged = true;
        mat4 colorMatrix;

        void traverseInZOrder(const LayerVector::Visitor& visitor) const;
        void traverseInReverseZOrder(const LayerVector::Visitor& visitor) const;
    };
```

9.SF.cpp中的`handleTransactionLocked`函数会根据`eTraversalNeeded`标志决定是否要检查所有的Layer对象。如果某个Layer对象有这个标志，将会调用他的`doTransaction`函数。`Layer::doTransaction`函数返回的flags如果有`eVisibleRegion`说明这个Layer需要更新，就把`mVisibleRegionDirty`设置为true。

```cpp
void SurfaceFlinger::handleTransactionLocked(uint32_t transactionFlags)
{
    // Notify all layers of available frames
    mCurrentState.traverseInZOrder([](Layer* layer) {
        layer->notifyAvailableFrames();
    });

    /*
     * Traversal of the children
     * (perform the transaction for each of them if needed)
     */

    if (transactionFlags & eTraversalNeeded) {
        mCurrentState.traverseInZOrder([&](Layer* layer) {
            uint32_t trFlags = layer->getTransactionFlags(eTransactionNeeded);
            if (!trFlags) return;

            const uint32_t flags = layer->doTransaction(0);
            if (flags & Layer::eVisibleRegion)
                mVisibleRegionsDirty = true;
        });
    }
    
    ......//这部分代码是根据每种显示设备的不同，设置和显示设备关联在一起的Layer（主要看LayerStack是否和DisplayDevice的layerStack相同）的TransformHint（主要指设备的显示方向orientation）

    commitTransaction();

    updateCursorAsync();
}
```

**Note:** `handleTransaction`的作用是处理系统在两次刷新期间的各种变化。Surfacelinger模块中不管是SurfaceFlinger类和Layer类

***

# 2. 硬件合成HWC2概述

Hardware Composer HAL(HWC)是指硬件完成图像数据组合并显示的能力。

SurfaceFlinger是一个系统服务（系统启动时启动），作用是接收来自多个源的Buffer数据，并进行合成，然后发送到显示设备进行显示。

SurfaceFlinger和HWC的相互配合，实现Android系统的合成和显示（非GPU合成）。

***

Android 7.0包含新版本的HWC（HWC2），Android需要自行配置。

Android 8.0，HWC2正式开启，并且版本升级为2.1。（`/frameworks/native/services/surfaceflinger/DisplayHardware/`）

HWC2是SurfaceFlinger用来与专门的窗口合成硬件进行通信（Device合成方式）。SurfaceFlinger包含使用3D图形处理器（GPU）执行窗口合成任务的备用途径，但是此路径并不理想（GPU合成方式），因为：
1. 通常，GPU没有针对此进行优化，因此能耗可能大于执行合成所需的能耗；
2. 每次SUrfaceFlinger使用GPU合成时，应用都无法使用处理器进行自我渲染，因此应尽可能使用专门的硬件而不是GPU进行合成。

GPU（Client合成）和HWC（Client合成）两种方式对比：

合成类型|耗电情况|性能情况|Alpha处理|DRM内容处理|其他限制
:-:|:-:|:-:|:-:|:-:|:-:
Device合成（HWC）|耗电低|性能高|很多Vendor的HWC不支持Alpha的处理和合成|基本都能访问DRM内容|能合成的Surface层数有限，对每种Surface类型处理层数有限
Client合成（GPU）|耗电高|性能低|能处理每个像素的Alpha及每个Layer的Alpha|早期版本GPU不能访问DRM的内容|目前的处理层数没有限制

**Note:**
1. Alpha处理： 图片的透明度（0～255或者0.0f~1.0f），数值越小透明度越高
2. DRM内容处理：（Digital Rights Management）一种业界使用广泛的数字内容版权保护技术。

***

## 2.1. HWC常规准则

Hardware Composer抽象层后的物理显示设备硬件可因设备而异。但是一般来说，遵循以下规则：
1. HWC应至少支持4个叠加层（状态栏、系统栏、应用、壁纸/背景）
2. 层可以大于屏幕，因此HWC应能处理大于显示屏的层（例如壁纸）
3. 应该同时支持预乘每个像素Alpha混合和每个平面Alpha混合
4. HWC应能够处理GPU、Camera、视频解码器（Video Decoder）生成的相同缓冲区，因此支持以下某些属性会很有帮助：
+ + RGBA打包顺序
+ + YUV格式
+ + Tiling,swizzling和步幅属性
5. 为了支持受保护的内容（Secure layer），必须提供受保护视频播放的硬件路径

**Note：**
1. RGBA是一种颜色值
2. YUV是一种颜色编码格式，可以说YUV流媒体是原始流数据，大部分的视频领域都在使用。他与RGB类似，但RGB更多的用于渲染时，而YUV则用在数据传输，因为它占用更少的频宽。当然，实时通讯为了降低带宽都会采用H264/H265编码。从字面意思理解，YUV的含义:Y代表亮度信息（灰度），UV分别代表色彩信息。YUV的常用名称有许多，如YUV422这是大部分镜头出来的数据，还有许多（yuv420,yuv444等）
3. Tiling简单来说就是将image进行切割，切成`M * N`小块，最后用的时候再进行拼接，类似铺瓷砖
4. swizzling是一种拌和技术，这是向量的单元可以被任意的重新排放或重复

HWC专注于优化，智能的选择要发送到叠加硬件的Surface，以最大限度减轻GPU的负载。**另一种优化是检测屏幕是否正在更新；如果不是，这将合成委托给OpenGL而不是HWC，以节省电量。但屏幕再次更新时，继续将合成分发给HWC**。

**为常见的用例做准备，比如：**
+ 纵向和横向模式下的全屏游戏
+ 带着字幕和播放控件的全屏视频
+ 主屏幕（状态栏、系统栏目、应用、动态壁纸）
+ 受保护的视频播放
+ 多显示设备支持

***

## 2.2. HWC2框架

从Android 8.0开始的Treble项目，对Android架构做了调整，让制造商以更低的成本更加轻松快速的将设备更新到Android系统。这就对HAL层有了很大的调整，利用提供给Vendor的接口，将Vendor的实现和Android上层分离开来。

这样的架构也使得HWC架构变得复杂，HWC属于`Binderized`的HAL类型。`Binderized`类型的HAL将上层Android和底层HAL分别采用两个不同的进程实现，中间采用Binder进行通信，为了和前面的Binder进行区别，这里采用`HWBinder`。

**可以将HWC分为以下几个部分：**

+ Binder 1：
+ + SurfaceFlinger Service
+ + HWC2 Client

+ Binder 2：
+ + HWC2 Server
+ + HWC2 Vendor Impl

**具体解释：**

1.Client端：Client就是指SurfaceFlinger。不过SurfaceFlinger采用前后端设计，以后和HWC相关的逻辑应该会放到后端（`SurfaceFlingerBE`），即`/frameworks/native/services/surfaceflinger/`
2.HWC Client端： 这一部分属于SurfaceFlinger进程，直接用过Binder通信，和HWC2的HAL Server交互。在SurfaceFlinger中采用`namespace HWC2`的命名空间，即`frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp`。
3.HWC2 Server端: 这里将建立一个进程实现HWC的`Server端`。服务端再调用底层Vendor的具体实现。并且，对于底层合成的实现不同，此处会做一些适配（`适配HWC1.x`），和FrameBuffer的实现。这部分包含三部分：**接口、实现、服务**，以动态库的形式存在：（`hardware/interfaces/graphics/composer/2.1/default/`）

+ + android.hardware.graphics.composer@2.1.so
+ + android.hardware.graphics.composer@2.1-impl.so
+ + android.hardware.graphics.composer@2.1-service.so

4.HWC Vendor实现： 这部分是HWC的具体实现，由硬件厂商完成，（`例如高通QCOM`），代码一般是`hardware/qcom/display/`。HWC必须采用`Binderized HAL`模式，但是不一定要实现HWC2的HAL版本。HWC2的实现需要配置，以Android 8.0为例，包含：

+ + 添加宏定义`TARGET_USERS_HWC2`
+ + 编译打包HWC2相关的so库
+ + SELinux相关的权限添加
+ + 配置`manifest.xml`：

```cpp
<hal format="hidl">
        <name>android.hardware.graphics.composer</name>
        <transport>hwbinder</transport>
        <version>2.1</version>
        <interface>
            <name>IComposer</name>
            <instance>default</instance>
        </interface>
</hal>
```

## 2.3. HWC2数据结构

HWC2的一些常用接口定义在头文件`hardware/libhardware/include/hardware/hwcomposer2.h`中，一些共用的数据定义是HAL的接口中:
+ `hardware/interfaces/graphics/common/1.0/`
+ `hardware/interfaces/graphics/composer/2.1/`

## 2.4. 图层Layer

每个Layer都有一组属性，用来定义和其他Layer的交互方式。他在每一个模块（层）代码定义的实现不一样，但是Layer的理念是一样的。

+ [SurfaceFlinger中](http://aosp.opersys.com/xref/android-10.0.0_r14/xref/frameworks/native/services/surfaceflinger/)

```shell
frameworks/native/services/surfaceflinger
├── Layer.h
├── Layer.cpp
├── ColorLayer.h
├── ColorLayer.cpp
├── BufferLayer.h
└── BufferLayer.cpp
|__ ...
```

+ HWC2中

```shell
frameworks/native/services/surfaceflinger/DisplayHardware
├── HWC2.h
└── HWC2.cpp
```

+ 在HAL中实现时，定义为hwc2_layer_t，是在头文件hwcomposer2.h中定义的:`typedef uint64_t hwc2_layer_t;`
+ HIDL中定义为Layer，这个Layer和`hwc2_layer_t`是一样的：`typedef uint64_t Layer;`

***

## 2.5. Layer按照类型划分

大致分为`BufferLayer`和`COlorLayer`（在SF中createLayer中），`BufferLayer`就是有Buffer的Lyaer（Bufferueue，GraphicsBuffer），需要上层应用Producer生长；`ColorLayer`可以绘制一种制定的颜色和透明度Alpha（取代之前的Dim Layer）。

***

## 2.6. Layer按照数据划分

大致分为`RGB Layer`和`YUV Layer`，前者是RGB格式，比较常见的就是UI界面的数据；后者的Buffer是YUV类型的，平常播放Video，Camera预览等，都是YUV类型的。

***

## 2.7. Layer属性*

> Layer的属性定义他和其他模块（层）的关系，和显示屏（DeisplayDevice）的关系等。Layer包含的属性类别如下（上述也有部分内容）：

***

## 2.8. 位置属性

> 定义层在其显示设备上的现实位置，包含层边缘的位置和其相对于其他层的Z-Order等，并且还定义了很多个**区域Region**：

```cpp
//frameworks/native/services/surfaceflinger/Layer.h
class Layer : public virtual RefBase {
    ... ...
public:
    ... ...
    // regions below are in window-manager space
    Region visibleRegion;
    Region coveredRegion;
    Region visibleNonTransparentRegion;
    Region surfaceDamageRegion;
```

Region中是很多个Rect的集合，即一个Layer的visibleRegion可能是几个Rect的集合（rect对象用来存储一个矩形框的左上角坐标、宽度和高度。描述矩形的宽度、高度和原点）

SurfaceFlinger中定义的Region都是从上层（WMS）传递过来的。而在HWC中，是用的下面的结构描述：

```cpp
//hardware/libhardware/include/hardware/hwcomposer_defs.h
typedef struct hwc_color {
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;
} hwc_color_t;

typedef struct hwc_float_color {
    float r;
    float g;
    float b;
    float a;
} hwc_float_color_t;

typedef struct hwc_frect {
    float left;
    float top;
    float right;
    float bottom;
} hwc_frect_t;

typedef struct hwc_rect {
    int left;
    int top;
    int right;
    int bottom;
} hwc_rect_t;

typedef struct hwc_region {
    size_t numRects;
    hwc_rect_t const* rects;
} hwc_region_t;
```

+ Transform，这个在SurfaceFlinger中定义的一个重要的结构，意思是变换矩阵，是一个`3*3`的矩阵。

**联系流程：**
`Rect <- Region <- Layer <- State <- Geometry <- Transform <- mat33`

+ Layer的两个状态：mCurrentState和mDrawingState，前者是给SurfaceFlinger的前段准备数据，后者是将数据给到合成。每个状态有两个Geometry的描述`request`（上层请求的）和`active`（当前正在使用的）。每个Geometry中有一个Transform矩阵，一个Transform包含一个mat33的整列。

Transform中包含两部分，一部分是位置Postion，另一部分是真正的2D的变换矩阵。通过下面两个函数设置：（对应Layer中的setPostion和setMatrix函数，这是上层WMS设置下来的）

```cpp
//frameworks/native/libs/ui/Transform.cpp
void Transform::set(float tx, float ty)
{
    mMatrix[2][0] = tx;
    mMatrix[2][1] = ty;
    mMatrix[2][2] = 1.0f;

    if (isZero(tx) && isZero(ty)) {
        mType &= ~TRANSLATE;
    } else {
        mType |= TRANSLATE;
    }
}

void Transform::set(float a, float b, float c, float d)
{
    mat33& M(mMatrix);
    M[0][0] = a;    M[1][0] = b;
    M[0][1] = c;    M[1][1] = d;
    M[0][2] = 0;    M[1][2] = 0;
    mType = UNKNOWN_TYPE;
}
......
```

***

## 2.9. 内容属性

> 定义显示的内容如何呈现（即Buffer）。Layer的显示，除了之前的几个区域Region描述，还有很多结构进一步描述才能显示，例如裁减（用来扩展内容的一部分以填充层的边界）和转换（用来显示旋转或者翻转的内容）等信息。HWCInfo结构体中包括了一些这样的信息：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngine/include/compositionengine/impl/OutputLayerCompositionState.h
struct OutputLayerCompositionState {
    // The region of this layer which is visible on this output
    Region visibleRegion;

    // If true, client composition will be used on this output
    bool forceClientComposition{false};

    // If true, when doing client composition, the target may need to be cleared
    bool clearClientTarget{false};

    // The display frame for this layer on this output
    Rect displayFrame;

    // The source crop for this layer on this output
    FloatRect sourceCrop;

    // The buffer transform to use for this layer o on this output.
    Hwc2::Transform bufferTransform{static_cast<Hwc2::Transform>(0)};

    // The Z order index of this layer on this output
    uint32_t z;

    /*
     * HWC state
     */

    struct Hwc {
        explicit Hwc(std::shared_ptr<HWC2::Layer> hwcLayer) : hwcLayer(hwcLayer) {}

        // The HWC Layer backing this layer
        std::shared_ptr<HWC2::Layer> hwcLayer;

        // The HWC composition type for this layer
        Hwc2::IComposerClient::Composition hwcCompositionType{
                Hwc2::IComposerClient::Composition::INVALID};

        // The buffer cache for this layer. This is used to lower the
        // cost of sending reused buffers to the HWC.
        HwcBufferCache hwcBufferCache;
    };

    // The HWC state is optional, and is only set up if there is any potential
    // HWC acceleration possible.
    std::optional<Hwc> hwc;

    // Debugging
    void dump(std::string& result) const;
};

```

## 2.10. 关系图

![Layer显示结构图](../../assets/post/2019/2019-12-22-android_HWC2/Layer_Struct.png)

**解释：**
1. Layer区域和屏幕区域，就是Layer和屏幕本身的大小区域
2. sourceCrop：剪切区域，sourceCrop是对Layer进行剪切的，值截取部分Layer的内容进行显示；sourceCrop不超过Layer的大小，超过没有意义。
3. displayFrame：显示区域，displayFrame表示Layer在屏幕上的显示区域，具体说来，是sourceCrop区域在显示屏上的显示区域。displayFrame一般来说，小于屏幕的区域。而displayFrame可能比sourceCrop大，可能小，这都是正常的，只是需要做缩放，这就是合成时需要处理的。
4. visibleRegion：可见区域，displayFrame 区域不一定都能看到的，如果存在上层Layer，那么displayFrame区域可能部分或全部被盖住，displayFrame没有被盖住的部分就是可见区域visibleRegion。
5. damageRegion 受损区域，或者称之为更新区域。damageRegion表示Layer内容被破坏的区域，也就是说这部分区域的内容变了，所以这个属性一般是和上一帧相比时才有意义。这算是对合成的一种优化，重新合成时，我们只去合成damageRegion区域，其他的可见区域还是用的上一帧的数据。
6. visibleNonTransparentRegion：可见非透明区域。透明区域transparentRegion是可见区域visibleRegion的一部分，只是这一部分透明的看到的是底层Layer的内容。在SurfaceFlinger的Layer中定义visibleNonTransparentRegion，表示可见而又不透明的部分。
7. coveredRegion：被覆盖的区域。表示Layer被TopLayer覆盖的区域，一看图就很好理解。从图中，你可以简单的认为是displayFrame和TopLayer区域重合的部分。

**注意：** 这里之所以说简单的认为，这是因为HWC空间的区域大小是SurfaceFlinger空间的区域经过缩放，经过Transform旋转，移动等后才得出的，要是混淆了就理解不对了。

***

## 2.11. 合成属性（确认用哪种合成方式）

> 定义层应如何与其他层合成。包括混合模式和用于Alpha合成的全层Alpha值等信息。总的说来，合成分为两个大类：`GPU合成`和`HWC合成`。根据具体的情况，分为下列几类：

```cpp
//hardware/libhardware/include/hardware/hwcomposer2.h
enum class Composition : int32_t {
    Invalid = HWC2_COMPOSITION_INVALID,
    Client = HWC2_COMPOSITION_CLIENT,
    Device = HWC2_COMPOSITION_DEVICE,
    SolidColor = HWC2_COMPOSITION_SOLID_COLOR,
    Cursor = HWC2_COMPOSITION_CURSOR,
    Sideband = HWC2_COMPOSITION_SIDEBAND,
};
```

**释义：**

1. Client 相对HWC2硬件合成的概念，主要是处理BufferLayer数据，用GPU处理。
2. Device HWC2硬件设备，主要处理BufferLayer数据，用HWC处理
3. SolidColor 固定颜色合成，主要处理ColorLayer数据，用HWC处理或GPU处理。
4. Cursor 鼠标标识合成，主要处理鼠标等图标，用HWC处理或GPU处理
5. Sideband Sideband为视频的边频带，一般需要需要硬件合成器作特殊处理，但是也可以用GPU处理。

在合成信息HWCInfo中，包含成的类型。通过Layer的setCompositionType方法进行指定：

```cpp
//frameworks/native/services/surfaceflinger/Layer.cpp
void Layer::setCompositionType(int32_t hwcId, HWC2::Composition type, bool callIntoHwc) {
    if (getBE().mHwcLayers.count(hwcId) == 0) {
        ALOGE("setCompositionType called without a valid HWC layer");
        return;
    }
    auto& hwcInfo = getBE().mHwcLayers[hwcId];
    auto& hwcLayer = hwcInfo.layer;
    ALOGV("setCompositionType(%" PRIx64 ", %s, %d)", hwcLayer->getId(), to_string(type).c_str(),
          static_cast<int>(callIntoHwc));  //默认true
    if (hwcInfo.compositionType != type) {
        ALOGV("    actually setting");
        hwcInfo.compositionType = type;
        if (callIntoHwc) {
            auto error = hwcLayer->setCompositionType(type);  //合成方式
            ALOGE_IF(error != HWC2::Error::None,
                     "[%s] Failed to set "
                     "composition type %s: %s (%d)",
                     mName.string(), to_string(type).c_str(), to_string(error).c_str(),
                     static_cast<int32_t>(error));
        }
    }
}
```

**确定合成类型分成三步：**
1. SurfaceFlinger制定合成类型，此时`callIntoHwc=true`，将类型制定给HWC
2. HWC根据实际情况看SurfaceFlinger制定的合成类型是否可以执行，如果不满足，作出修改
3. SurfaceFlinger根据HWC的修改情况再作出调整，最终确认合成类型，此时`callIntoHwc=false`

## 2.12. 优化属性

> 提供一些非必须的参数，以供HWC进行合成的优化。包括层的可见区域以及层的哪个部分自上一帧以来已经更新等信息。也就是前面说到的visibleRegion，damageRegion等。

***

# 3. 小结

> 本篇主要是SurfaceFlinger概述，和HWC2的概述，还有Layer的属性和类型，合成方式的内容。  
> 另外还有关于HWC的内容，和Display显示设备的信息重新划分单独的一篇学习。

***

# 4. 参考

+ 转载夕月风大佬博客： https://www.jianshu.com/p/824a9ddf68b9