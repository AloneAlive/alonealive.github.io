---
layout: single
related: false
title:  Android HWC硬件合成
date:   2020-04-01 23:32:00
categories: android
tags: android graphics
toc: true
---

> 参考Android P AOSP源码，对HWC合成分析了解

<!--more-->

# 1. 显示屏Display

> 显示屏Display是合成的另一个重要单元，系统可以有多个显示设备，并且在正常系统操作期间可以添加/删除显示设备。该添加/删除操作可以对应HWC设备的热插拔请求，或者应客户端的请求进行，这允许创建虚拟显示设备，其内容会渲染到离屏缓冲区（而不是物理显示设备）。

> 可以通过Dump SF查看display的layer信息，同时也可以根据layerstack异同判断多个display是否用的同一个layer。

HWC中，SurfaceFlinger中创建的Layer，在合成开始时，将被指定到每个Display上，此后合成过程中，每个Display合成指定给自己的Layer。

SurfaceFlinger前端，每个显示屏用`DisplayDevice`类描述，在后端显示数据用`DisplayData`描述。而在HWC2的Client端，定义了Display类进行描述。对于HWC2服务端则用`hwc2_display_t`描述，他只是一个序号，Vendor具体实现时，才具体的去管理Display的信息。

HWC2提供相应函数来确定给定显示屏的属性，在不同配置（例如4K/1080分辨率）和颜色模式（例如Native颜色或者真彩sRGB）之间切换，以及打开/关闭显示设备或者将其切换到低功率模式（如果支持）。

# 2. HWC设备composerDevice

**Note:**
注意显示屏Display和合成设备的区别，HWC合成设备只有一个，定义在头文件：

```cpp
//hardware/libhardware/include/hardware/hwcomposer2.h
typedef struct hwc2_device {
    struct hw_device_t common;

    void (*getCapabilities)(struct hwc2_device* device, uint32_t* outCount,
            int32_t* /*hwc2_capability_t*/ outCapabilities);

    hwc2_function_pointer_t (*getFunction)(struct hwc2_device* device,
            int32_t /*hwc2_function_descriptor_t*/ descriptor);
} hwc2_device_t;
```

## 2.1. HWC合成服务

`hardware/interfaces/graphics/composer/2.1/default`这个HWC的的默认服务。SurfaceFlinger初始化时，可以通过属性`debug.sf.hwc_service_name`来制定，默认为default。在编译时，manifest.xml中配置的也是default。

```cpp 
//frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
std::string getHwcServiceName() {
    char value[PROPERTY_VALUE_MAX] = {};
    property_get("debug.sf.hwc_service_name", value, "default");
    ALOGI("Using HWComposer service: '%s'", value);
    return std::string(value);
}
```

HWC服务分两部分：

1. 可以执行程序`android.hardware.graphics.composer@2.1-service`（在目录/vendor/bin/hw/）

其main函数如下，通过defaultPassthroughServiceImplementation函数注册IComposer服务。

```cpp
//hardware/interfaces/graphics/composer/2.1/default/service.cpp
int main() {
    // the conventional HAL might start binder services
    android::ProcessState::initWithDriver("/dev/vndbinder");
    android::ProcessState::self()->setThreadPoolMaxThreadCount(4);
    android::ProcessState::self()->startThreadPool();

    // same as SF main thread
    struct sched_param param = {0};
    param.sched_priority = 2;
    if (sched_setscheduler(0, SCHED_FIFO | SCHED_RESET_ON_FORK,
                &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO: %d", errno);
    }

    return defaultPassthroughServiceImplementation<IComposer>(4);
}
```

对应.rc文件：

```s
service vendor.hwcomposer-2-1 /vendor/bin/hw/android.hardware.graphics.composer@2.1-service
    class hal animation
    user system
    group graphics drmrpc
    capabilities SYS_NICE
    onrestart restart surfaceflinger
```

2. 实现库`android.hardware.graphics.composer@2.1-impl.so`

hwc的执行程序中，注册的IComposer，将调到对应的FETCH函数，FETCH函数实现及是so库中。FETCH如下：

```cpp
IComposer* HIDL_FETCH_IComposer(const char*)
{
    const hw_module_t* module = nullptr;
    int err = hw_get_module(HWC_HARDWARE_MODULE_ID, &module);
    if (err) {
        ALOGI("falling back to FB HAL");
        err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    }
    if (err) {
        ALOGE("failed to get hwcomposer or fb module");
        return nullptr;
    }

    return new HwcHal(module);
}
```

FETCH函数中，才正在去加载Vendor的实现，通过统一的接口`hw_get_module`根据IDHWC_HARDWARE_MODULE_ID去加载。加载完成后，创建HAL描述类似HwcHal。

```cpp
HwcHal::HwcHal(const hw_module_t* module)
    : mDevice(nullptr), mDispatch(), mMustValidateDisplay(true), mAdapter() {
    uint32_t majorVersion;
    if (module->id && strcmp(module->id, GRALLOC_HARDWARE_MODULE_ID) == 0) {
        majorVersion = initWithFb(module);
    } else {
        majorVersion = initWithHwc(module);
    }

    initCapabilities();
    if (majorVersion >= 2 && hasCapability(HWC2_CAPABILITY_PRESENT_FENCE_IS_NOT_RELIABLE)) {
        ALOGE("Present fence must be reliable from HWC2 on.");
        abort();
    }

    initDispatch();
}
```

如果是FrameBuffer驱动，通过initWithFb初始化。如果是HWC驱动，通过initWithHwc初始化。我们需要的是HWC2的接口，如果不是HWC2的HAl实现，那么需要做适配。

***

# 3. Client和Server的通信

SurfaceFlinger和HWC服务之间，很多函数，并没有直接的调用，而是通过Buffer的读写来实现调用和参数的传递的。所以，Client端和Server端通信，基本通过以下相关的途径：

+ 通过IComposerClient.hal接口
+ 通过IComposer.hal接口
+ 通过command Buffer

Server端回调Client端，通过IComposerCallback.hal接口。

hal接口的方式，其本质就是Binder。又加了一个command Buffer的方式，其实这是为了解决Binder通信慢的问题。

## 3.1. HWC2中Fence的更改

HWC 2.0 中同步栅栏的含义相对于以前版本的HAL已有很大的改变。

在 HWC v1.x 中，释放Fence和退出Fence是推测性的。在帧 N 中检索到的Buffer的释放Fence或显示设备的退出Fence不会先于在帧 N + 1 中检索到的Fence变为触发状态。换句话说，该Fence的含义是“不再需要您为帧 N 提供的Buffer内容”。这是推测性的，因为在理论上，SurfaceFlinger 在帧 N 之后的一段不确定的时间内可能无法再次运行，这将使得这些栅栏在该时间段内不会变为触发状态。

在 HWC 2.0 中，释放Fence和退出Fence是非推测性的。在帧 N 中检索到的释放Fence或退出Fence，将在相关Buffer的内容替换帧 N - 1 中缓冲区的内容后立即变为触发状态，或者换句话说，该Fence的含义是“您为帧 N 提供的缓冲区内容现在已经替代以前的内容”。这是非推测性的，因为在硬件呈现此帧的内容之后，该栅栏应该在 presentDisplay 被调用后立即变为触发状态。

**Notes:** 关于Fence同步机制需要单独拎出来梳理学习。

***

# 4. 参考

+ 转载夕月风大佬博客： https://www.jianshu.com/p/824a9ddf68b9
+ 参考源码： http://aosp.opersys.com/xref/android-10.0.0_r14/