---
layout: single
related: false
title:  Android Q SurfaceFlinger合成（二）
date:   2020-10-31 18:42:00
categories: android
tags: android graphics
toc: true
---

> 继上篇《Android　Ｑ SurfaceFlinger合成（一）》中SF对INVALIDATE信息处理，针对Layer属性变化、显示设备变化等情况处理，将mCurrentState提交到mDrawingState。然后遍历mDrawingState的Layer，将新的Buffer内容更新绑定到Layer纹理对象。经过这些流程，决定是否需要SF进行合成刷新，如果需要则调用`handleMessageRefresh`开始合成处理。

<!--more-->

# 1. signalRefresh

在onMessageReceivedINVALIDATE信息处理完成后，如果需要刷新，则会触发刷新：

```cpp
//SF.cpp
void SurfaceFlinger::onMessageReceived(int32_t what) NO_THREAD_SAFETY_ANALYSIS {
    ATRACE_CALL();
    switch (what) {
        case MessageQueue::INVALIDATE: {
            ......
            refreshNeeded |= mRepaintEverything;
            //在BootStage:：BOOTLOADER中时不要调用signalRefresh，不想用一个空白屏幕代替bootloader引导加载程序启动。
            //这样可以节省HWC不必要的工作
            if (refreshNeeded && CC_LIKELY(mBootStage != BootStage::BOOTLOADER)) {
                //如果事务修改了窗口状态，新的Buffer被获取到，或者HWC已经请求一个新的repaint，则发出刷新信号
                signalRefresh();
            }
            break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
    }
}

void SurfaceFlinger::signalRefresh() {
    mRefreshPending = true;
    mEventQueue.refresh();
}
```

**需要刷新的情况：**

1. 有新的Transaction处理
2. PageFlip时，有Buffer更新
3. 有重新合成请求时mRepaintEverything，这是响应HWC的请求时触发的。

## 1.1. MessageQueue分发refresh

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::refresh() {
    mHandler->dispatchRefresh();
}

void MessageQueue::Handler::dispatchRefresh() {
    if ((android_atomic_or(eventMaskRefresh, &mEventMask) & eventMaskRefresh) == 0) {
        mQueue.mLooper->sendMessage(this, Message(MessageQueue::REFRESH));
    }
}

//最终回调handleMessage，处理REFRESH的message
//这过程中不用去等Vsync的，INVALIDATE时，是需要等Vsync的
//即INVALIDATE和REFRESH是在同一个Vsync周期内完成的
void MessageQueue::Handler::handleMessage(const Message& message) {
    switch (message.what) {
        case INVALIDATE:
            android_atomic_and(~eventMaskInvalidate, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
        case REFRESH:
            android_atomic_and(~eventMaskRefresh, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
    
```

# 2. handleMessageRefresh刷新总流程

handleMessageRefresh函数包含了刷新（合成）一帧显示数据的所有流程。

```cpp
//SF.cpp
void SurfaceFlinger::onMessageReceived(int32_t what) NO_THREAD_SAFETY_ANALYSIS {
    ATRACE_CALL();d
    switch (what) {
        case MessageQueue::INVALIDATE: {
            ......
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
    }
}

void SurfaceFlinger::handleMessageRefresh() {
    ATRACE_CALL();

    mRefreshPending = false;
    const bool repaintEverything = mRepaintEverything.exchange(false);
    //合成前预处理工作
    preComposition();
    //计算和存储每个Layer的脏区域
    rebuildLayerStacks();
    //
    calculateWorkingSet();//和P不同
    //遍历Display
    for (const auto& [token, display] : mDisplays) {
        beginFrame(display);//和P不同
        prepareFrame(display); //和P不同
        doDebugFlashRegions(display, repaintEverything);
        //先进行GL合成，将合成后的的图像放在HWC任务列表的最后为止
        //然后由HWC进行合成并输出到屏幕
        doComposition(display, repaintEverything);
    }

    logLayerStats();

    postFrame();
    //合成善后工作
    postComposition();

    mHadClientComposition = false;
    mHadDeviceComposition = false;
    for (const auto& [token, displayDevice] : mDisplays) {
        auto display = displayDevice->getCompositionDisplay();
        const auto displayId = display->getId();
        mHadClientComposition =
                mHadClientComposition || getHwComposer().hasClientComposition(displayId);
        mHadDeviceComposition =
                mHadDeviceComposition || getHwComposer().hasDeviceComposition(displayId);
    }

    mVsyncModulator.onRefreshed(mHadClientComposition);
    mLayersWithQueuedFrames.clear();
}
```

## 2.1. preComposition合成前预处理

```cpp
//SF.cpp
void SurfaceFlinger::preComposition()
{
    ATRACE_CALL();
    ALOGV("preComposition");

    mRefreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);

    bool needExtraInvalidate = false;
    //遍历所有需要进行合成的Layer（mDrawingState）
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        //调用onPreComposition，但绘制
        if (layer->onPreComposition(mRefreshStartTime)) {
            needExtraInvalidate = true;
        }
    });
    //通过上述函数的返回值来判断是否需要再次触发SurfaceFlinger接受Vsync合成
    if (needExtraInvalidate) {
        signalLayerUpdate();
    }
}
```

其中`onPreComposition`函数的返回值针对不同Layer：

+ ColorLayer和ContainLayer固定返回false
+ BufferLayer如下：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayer.cpp
bool BufferLayer::onPreComposition(nsecs_t refreshStartTime) {
    if (mBufferLatched) {
        Mutex::Autolock lock(mFrameEventHistoryMutex);
        //mFrameEventHistory记录PreComposition事件
        mFrameEventHistory.addPreComposition(mCurrentFrameNumber, refreshStartTime);
    }
    mRefreshPending = false;
    return hasReadyFrame();
}

bool BufferLayer::hasReadyFrame() const {
    return hasFrameUpdate() || getSidebandStreamChanged() || getAutoRefresh();
}

//****** BufferQueueLayer.cpp **********
bool BufferQueueLayer::hasFrameUpdate() const {
    //之前在acquireBuffer的时候已经做了-1操作，而此处是现在BufferQueue中还有Buffer
    //（即仍有待处理的Buffer，就需要下次Vsync到来的时候再次执行合成）
    return mQueuedFrames > 0;
}

bool BufferQueueLayer::getSidebandStreamChanged() const {
    //SidebandStream改变
    return mSidebandStreamChanged;
}

bool BufferQueueLayer::getAutoRefresh() const {
    //自动刷新模式
    return mAutoRefresh;
}
```

***

## 2.2. rebuildLayerStacks重构Layer栈

执行该函数，将完成创建Layer栈。

此时需要进行合成显示的数据已经被更新到每个Display各自的`layersSortedByZ`中。

```cpp
//SF.cpp
void SurfaceFlinger::invalidateHwcGeometry()
{
    mGeometryInvalid = true;
}

void SurfaceFlinger::rebuildLayerStacks() {
    ATRACE_CALL();
    ALOGV("rebuildLayerStacks");

    // rebuild the visible layer list per screen
    // 前提是存在脏区域，即mVisibleRegionsDirty为true
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ATRACE_NAME("rebuildLayerStacks VR Dirty");
        mVisibleRegionsDirty = false;
        //重置mGeometryInvalid标记
        invalidateHwcGeometry();

        //遍历每个屏幕，因为每个Display是分开合成的
        //根据显示屏的特性，分别进行合成，合成后的数据也送给各自的显示屏
        //mDisplays是当前系统中的显示屏
        for (const auto& pair : mDisplays) {
            ......
            if (displayState.isEnabled) {
                //计算屏幕的脏区域、每个Layer的可见区域、被覆盖的区域、可见非透明区域（见下面一小节）
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);
                //正序遍历mDrawingState合成列表的layer
                //和当前的显示设备进行比较，Layer的脏区域是否在显示设备的显示区域内
                //如果在显示区域内的话说明该layer是需要更新的，则更新到显示设备的`VisibleLayersSortedByZ`列表中，等待被合成
                mDrawingState.traverseInZOrder([&](Layer* layer) {
                    //
                    auto compositionLayer = layer->getCompositionLayer();
                    if (compositionLayer == nullptr) {
                        return;
                    }

                    const auto displayId = displayDevice->getId();
                    sp<compositionengine::LayerFE> layerFE = compositionLayer->getLayerFE();
                    LOG_ALWAYS_FATAL_IF(layerFE.get() == nullptr);

                    bool needsOutputLayer = false;
                    //计算Layer需要绘制的区域drawRegion
                    if (display->belongsInOutput(layer->getLayerStack(),
                                                 layer->getPrimaryDisplayOnly())) {
                        Region drawRegion(tr.transform(
                                layer->visibleNonTransparentRegion));
                        //将Layer的可见区域和Display大小做交集
                        drawRegion.andSelf(bounds);
                        if (!drawRegion.isEmpty()) {
                            needsOutputLayer = true;
                        }
                    }
                    //drawRegion不为空，将该Layer加到当前Display的Layer列表中
                    if (needsOutputLayer) {
                        layersSortedByZ.emplace_back(
                                display->getOrCreateOutputLayer(displayId, compositionLayer,
                                                                layerFE));
                        deprecated_layersSortedByZ.add(layer);

                        auto& outputLayerState = layersSortedByZ.back()->editState();
                        outputLayerState.visibleRegion =
                                tr.transform(layer->visibleRegion.intersect(displayState.viewport));
                    //之前Layer可见，现在不可见，将销毁掉HWC Layer
                    } else if (displayId) {
                        //对于正在从HWC Display中移除的和已经排队的帧的那些Layer，将他们添加到一个发布的Layer列表中，以便可以设置一个fence
                        bool hasExistingOutputLayer =
                                display->getOutputLayerForLayer(compositionLayer.get()) != nullptr;
                        bool hasQueuedFrames = std::find(mLayersWithQueuedFrames.cbegin(),
                                                         mLayersWithQueuedFrames.cend(),
                                                         layer) != mLayersWithQueuedFrames.cend();

                        if (hasExistingOutputLayer && hasQueuedFrames) {
                            //销毁掉的Layer放置到layersNeedingFences中
                            //虽然不需要releaseFence，但是还是需要fence去释放旧的Buffer
                            layersNeedingFences.add(layer);
                        }
                    }
                }); //遍历Layer结束
            }
            //最后将数据更新到DisplayDevice中
            display->setOutputLayersOrderedByZ(std::move(layersSortedByZ));

            displayDevice->setVisibleLayersSortedByZ(deprecated_layersSortedByZ);
            displayDevice->setLayersNeedingFences(layersNeedingFences);

            Region undefinedRegion{bounds};
            undefinedRegion.subtractSelf(tr.transform(opaqueRegion));

            display->editState().undefinedRegion = undefinedRegion;
            display->editState().dirtyRegion.orSelf(dirtyRegion);
        }//遍历Display结束
    }
}
```

### 2.2.1. computeVisibleRegions计算可见区域

代码包含注解：

```cpp
//outOpaqueRegion是屏幕的非透明区域
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
                                           Region& outDirtyRegion, Region& outOpaqueRegion) {
    ATRACE_CALL();
    ALOGV("computeVisibleRegions");
    //获取当前合成显示屏的display
    auto display = displayDevice->getCompositionDisplay();
    //针对当前整个Display！
    //当前Layer上层所有Layer不透明区域的累加
    Region aboveOpaqueLayers;
    //当前Layer上层所有Layer可见区域的累加
    Region aboveCoveredLayers;
    //脏区域
    Region dirty;

    //清空屏幕脏区域（每个Layer脏区域的和），将计算好的区域值设置到Layer中
    outDirtyRegion.clear();
    //反序号遍历，即从最上面layer层开始遍历
    mDrawingState.traverseInReverseZOrder([&](Layer* layer) {
        // start with the whole surface at its current location
        const Layer::State& s(layer->getDrawingState());
        // layerStackId必须匹配
        if (!display->belongsInOutput(layer->getLayerStack(), layer->getPrimaryDisplayOnly())) {
            return;
        }
        //针对每个Layer！
        //完全不透明区域
        Region opaqueRegion;

        //可见区域（屏幕上可见的Surface区域，而且是不完全透明）
        //包含半透明区域：半透明Surface覆盖的区域被视为可见
        Region visibleRegion;

        //覆盖区域：被全部覆盖的Surface区域（包括被透明区域覆盖的区域）
        Region coveredRegion;

        //完全透明区域：Surface完全透明的部分，如果没有可见的非透明区域，这个Layer就可以从Layer列表中删除
        //并且不会影响该Layer本身或其下方layer的可见区域大小
        //这个区域可能不太准，因为有的APP不遵守SurfaceView的限制
        Region transparentRegion;

        // 通过将可见区域设置为空来处理隐藏的Surface
        if (CC_LIKELY(layer->isVisible())) {
            //isOpaque表示Lyaer是非透明的Layer（上层应用层设置）
            const bool translucent = !layer->isOpaque(s);
            //获取Layer在屏幕上的大小
            //注：activeWidth和activeHeight是Layer本身的大小,用win表示
            //crop是Layer的源剪截区域，由上层设置，表示该Layer只截取crop的区域进行合成显示。可能比win大也可能小，需要取交集，截取重复的部分
            Rect bounds(layer->getScreenBounds());
            //即上面返回Layer大小设置为可见区域，但是后续还会被裁剪
            visibleRegion.set(bounds);
            //Layer的变换矩阵，例如旋转，适配显示屏幕
            ui::Transform tr = layer->getTransform();
            //可见区域不为空（一般情况下，如果layer是非透明的，非透明区域就是可见区域）
            if (!visibleRegion.isEmpty()) {
                // 从可见区域移除完全透明区域
                if (translucent) {
                    if (tr.preserveRects()) {
                        // transform the transparent region
                        transparentRegion = tr.transform(layer->getActiveTransparentRegion(s));
                    } else {
                        //转型太复杂，不能做到透明区域优化
                        transparentRegion.clear();
                    }
                }
                //计算不透明区域
                const int32_t layerOrientation = tr.getOrientation();
                if (layer->getAlpha() == 1.0f && !translucent &&
                        layer->getRoundedCornerState().radius == 0.0f &&
                        ((layerOrientation & ui::Transform::ROT_INVALID) == false)) {
                    // the opaque region is the layer's footprint
                    opaqueRegion = visibleRegion;
                }
            }
        }
        ......
        // 将覆盖区域剪辑到可见区域
        //遍历时，第一层时，aboveCoveredLayers为空，coveredRegion也是为空，最上面一层是没有被覆盖的，当然为空
        coveredRegion = aboveCoveredLayers.intersect(visibleRegion);

        //为下一个（底）layer更新aboveCoveredLayers
        //更新aboveCoveredLayers，该层之下的Layer都被该层Layer覆盖，所以这里和可见区域做一个或操纵，最下面的区域被覆盖的越大
        aboveCoveredLayers.orSelf(visibleRegion);

        //减去在我aboveOpaqueLayers覆盖的不透明区域
        //可见区域要减掉该层之上的非透明区域
        visibleRegion.subtractSelf(aboveOpaqueLayers);

        // 计算Layer的脏区域
        // contentDirty表示包含脏区域内容，即Layer的可见区域被修改了
        if (layer->contentDirty) {
            //需要使整个地区无效
            dirty = visibleRegion;
            //以及旧的可见区域
            dirty.orSelf(layer->visibleRegion);
            layer->contentDirty = false;
        } else {
            //计算暴露出来的区域 exposedRegion
            //包含两部分：1.之前被覆盖的区域，现在可见了；2.现在暴露的比之前的少
            //注：1是从整体的可见区域开始，但是只保留以前被覆盖的区域（现在暴露了）；2是处理那种因为重新调整了大小从而暴露出来的区域
            const Region newExposed = visibleRegion - coveredRegion;
            const Region oldVisibleRegion = layer->visibleRegion;
            const Region oldCoveredRegion = layer->coveredRegion;
            const Region oldExposed = oldVisibleRegion - oldCoveredRegion;
            dirty = (visibleRegion&oldCoveredRegion) | (newExposed-oldExposed);
        }
        dirty.subtractSelf(aboveOpaqueLayers);

        //累计到屏幕脏区域
        outDirtyRegion.orSelf(dirty);

        //更新opaqueRegion到aboveOpaqueLayers，为下面（底）的Layer做准备
        aboveOpaqueLayers.orSelf(opaqueRegion);

        //在屏幕空间保存可见区域
        //设置可见区域
        layer->setVisibleRegion(visibleRegion);
        //设置被覆盖的区域
        layer->setCoveredRegion(coveredRegion);
        //设置可见的非透明区域（=可见区域-透明区域）
        layer->setVisibleNonTransparentRegion(
                visibleRegion.subtract(transparentRegion));
    });  //遍历结束
    //屏幕的非透明区域
    outOpaqueRegion = aboveOpaqueLayers;
}
```

***

## 2.3. calculateWorkingSet

在Android P中是用的`setUpHWComposer`函数，Q升级后将其分成几个单独的函数。以下是第一个：

```cpp
//SF.cpp
void SurfaceFlinger::calculateWorkingSet() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    //创建H/W工作列表（判断变量在rebuildLayerStacks中变动）
    if (CC_UNLIKELY(mGeometryInvalid)) {
        //重置
        mGeometryInvalid = false;
        //遍历Display
        for (const auto& [token, displayDevice] : mDisplays) {
            auto display = displayDevice->getCompositionDisplay();

            uint32_t zOrder = 0;
            //遍历Layer
            for (auto& layer : display->getOutputLayersOrderedByZ()) {
                //**调用CompositionEngine/src/OutputLayer.cpp返回mState
                auto& compositionState = layer->editState();
                //forceClientComposition指强制GPU合成（Client）
                //mDebugDisableHWC指开发者选项的“停用HWC叠加层”，mDebugRegion指调试Region
                compositionState.forceClientComposition = false;
                if (!compositionState.hwc || mDebugDisableHWC || mDebugRegion) {
                    compositionState.forceClientComposition = true;
                }

                //输出的z顺序值是一个简单的计数器设置
                compositionState.z = zOrder++;

                //更新Display自己的合成状态
                layer->getLayerFE().latchCompositionState(layer->getLayer().editState().frontEnd,
                                                          true);

                //重新计算输出output layer的合成状态
                layer->updateCompositionState(true);

                //写入到HWC，该函数会设置Layer的几何尺寸（见下一小节该函数释义）
                layer->writeStateToHWC(true);
            }
        }
    }
    //设置每层Layer的frame帧数据
    //遍历Display
    for (const auto& [token, displayDevice] : mDisplays) {
        auto display = displayDevice->getCompositionDisplay();
        const auto displayId = display->getId();
        if (!displayId) {
            continue;
        }
        auto* profile = display->getDisplayColorProfile();
        //设置颜色矩阵
        if (mDrawingState.colorMatrixChanged) {
            display->setColorTransform(mDrawingState.colorMatrix);
        }
        Dataspace targetDataspace = Dataspace::UNKNOWN;
        if (useColorManagement) {
            ColorMode colorMode;
            RenderIntent renderIntent;
            pickColorMode(displayDevice, &colorMode, &targetDataspace, &renderIntent);
            //设置色彩模式
            display->setColorMode(colorMode, targetDataspace, renderIntent);
        }
        //遍历可见layer
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            //根据layer的数据空间dataSpace（通过dump SurfaceFlinger可以查看到）来设置layer的合成方式
            if (layer->isHdrY410()) {
                layer->forceClientComposition(displayDevice);
            } else if ((layer->getDataSpace() == Dataspace::BT2020_PQ ||
                        layer->getDataSpace() == Dataspace::BT2020_ITU_PQ) &&
                       !profile->hasHDR10Support()) {
                layer->forceClientComposition(displayDevice);
            } else if ((layer->getDataSpace() == Dataspace::BT2020_HLG ||
                        layer->getDataSpace() == Dataspace::BT2020_ITU_HLG) &&
                       !profile->hasHLGSupport()) {
                layer->forceClientComposition(displayDevice);
            }

            if (layer->getRoundedCornerState().radius > 0.0f) {
                layer->forceClientComposition(displayDevice);
            }

            if (layer->getForceClientComposition(displayDevice)) {
                ALOGV("[%s] Requesting Client composition", layer->getName().string());
                //设置合成方式GPU合成
                layer->setCompositionType(displayDevice,
                                          Hwc2::IComposerClient::Composition::CLIENT);
                continue;
            }
            //设置每一层Layer的显示数据
            const auto& displayState = display->getState();
            layer->setPerFrameData(displayDevice, displayState.transform, displayState.viewport,
                                   displayDevice->getSupportedPerFrameMetadata(),
                                   isHdrColorMode(displayState.colorMode) ? Dataspace::UNKNOWN
                                                                          : targetDataspace);
        }
    }

    mDrawingState.colorMatrixChanged = false;

    for (const auto& [token, displayDevice] : mDisplays) {
        auto display = displayDevice->getCompositionDisplay();
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            auto& layerState = layer->getCompositionLayer()->editState().frontEnd;
            layerState.compositionType = static_cast<Hwc2::IComposerClient::Composition>(
                    layer->getCompositionType(displayDevice));
        }
    }
}
```

**颜色矩阵如下，可以在开发这选项中设置，有`模拟颜色空间`选项。其支持的transform主要有：**

```cpp
//system/core/libsystem/include/system/graphics-base-v1.0.h
typedef enum {
    HAL_COLOR_TRANSFORM_IDENTITY = 0,
    HAL_COLOR_TRANSFORM_ARBITRARY_MATRIX = 1,
    HAL_COLOR_TRANSFORM_VALUE_INVERSE = 2,
    HAL_COLOR_TRANSFORM_GRAYSCALE = 3,
    HAL_COLOR_TRANSFORM_CORRECT_PROTANOPIA = 4,
    HAL_COLOR_TRANSFORM_CORRECT_DEUTERANOPIA = 5,
    HAL_COLOR_TRANSFORM_CORRECT_TRITANOPIA = 6,
} android_color_transform_t;
```

### 2.3.1. writeStateToHWC设置Layer几何尺寸

OutputLayer.cpp是Android新分离出来的文件，writeStateToHWC函数也是分离成一个单独的函数，以供SurfaceFlinger的calculateWorkingSet函数调用。

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngine/src/OutputLayer.cpp
void OutputLayer::writeStateToHWC(bool includeGeometry) const {
    // Skip doing this if there is no HWC interface
    //此处的State数据是来源于DrawingState
    if (!mState.hwc) {
        return;
    }

    auto& hwcLayer = (*mState.hwc).hwcLayer;
    if (!hwcLayer) {
        ALOGE("[%s] failed to write composition state to HWC -- no hwcLayer for output %s",
              mLayerFE->getDebugName(), mOutput.getName().c_str());
        return;
    }

    if (includeGeometry) {
        //输出依赖状态 Output dependent state
        //计算DisplayFrame
        if (auto error = hwcLayer->setDisplayFrame(mState.displayFrame);
            error != HWC2::Error::None) {... //log打印
        }
        //计算SourceCrop
        if (auto error = hwcLayer->setSourceCrop(mState.sourceCrop); error != HWC2::Error::None) {
            ... //log打印
        }
        //设置zOrder
        if (auto error = hwcLayer->setZOrder(mState.z); error != HWC2::Error::None) {
            ... //log打印
        }
        //设置transform旋转
        if (auto error =
                    hwcLayer->setTransform(static_cast<HWC2::Transform>(mState.bufferTransform));
            error != HWC2::Error::None) {
            ... //log打印
        }

        //输出独立状态 Output independent state
        const auto& outputIndependentState = mLayer->getState().frontEnd;
        //设置图层混合模式（见https://developer.android.google.cn/reference/android/graphics/BlendMode）
        if (auto error = hwcLayer->setBlendMode(
                    static_cast<HWC2::BlendMode>(outputIndependentState.blendMode));
            error != HWC2::Error::None) {
            ... //log打印
        }
        //设置Alpha透明度
        if (auto error = hwcLayer->setPlaneAlpha(outputIndependentState.alpha);
            error != HWC2::Error::None) {
            ... //log打印
        }
        //设置Layer信息
        //type和appId是Android Framework层创建SurfaceControl时设置的
        //其中type包含ScreenshotSurface、Background等
        //appId是应用进程号
        if (auto error =
                    hwcLayer->setInfo(outputIndependentState.type, outputIndependentState.appId);
            error != HWC2::Error::None) {
            A... //log打印
        }
    }
}
```

其中`setBlendModes`设置混合模式（两个Layer直接的混合方式），主要有以下几种：

```cpp hardware/libhardware/include/hardware/hwcomposer2.h
/* Blend modes, settable per layer */
typedef enum {
    HWC2_BLEND_MODE_INVALID = 0, 

    /* colorOut = colorSrc */
    HWC2_BLEND_MODE_NONE = 1,  //不混合，源和输出不变

    /* colorOut = colorSrc + colorDst * (1 - alphaSrc) */
    HWC2_BLEND_MODE_PREMULTIPLIED = 2,  //预乘，Dst需要做Alpha的处理

    /* colorOut = colorSrc * alphaSrc + colorDst * (1 - alphaSrc) */
    HWC2_BLEND_MODE_COVERAGE = 3, //覆盖方式，源和Dst都需要做Alpha透明度的处理
} hwc2_blend_mode_t;
```

### 2.3.2. setPerFrameData设置每一层Layer显示数据

1. `ColorLayer::setPerFrameData`

```cpp
//frameworks/native/services/surfaceflinger/ColorLayer.cpp
void ColorLayer::setPerFrameData(const sp<const DisplayDevice>& display,
                                 const ui::Transform& transform, const Rect& viewport,
                                 int32_t /* supportedPerFrameMetadata */,
                                 const ui::Dataspace targetDataspace) {
    ......
    //设置可见区域，之前已经计算好，但是此处需要确保可见区域在Display的窗口内
    auto error = hwcLayer->setVisibleRegion(visible);
    ...
    //设置数据空间
    error = hwcLayer->setDataspace(dataspace);
    ...

    auto& layerCompositionState = getCompositionLayer()->editState().frontEnd;
    layerCompositionState.dataspace = mCurrentDataSpace;
    //设置RGB颜色，Alpha默认255（全透明）
    half4 color = getColor();
    error = hwcLayer->setColor({static_cast<uint8_t>(std::round(255.0f * color.r)),
                                static_cast<uint8_t>(std::round(255.0f * color.g)),
                                static_cast<uint8_t>(std::round(255.0f * color.b)), 255});
    ...
    layerCompositionState.color = {static_cast<uint8_t>(std::round(255.0f * color.r)),
                                   static_cast<uint8_t>(std::round(255.0f * color.g)),
                                   static_cast<uint8_t>(std::round(255.0f * color.b)), 255};
    //色彩ColorLayer不需要变换矩阵，清除掉
    error = hwcLayer->setTransform(HWC2::Transform::None);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to clear transform: %s (%d)", mName.string(), to_string(error).c_str(),
              static_cast<int32_t>(error));
    }
    ...
    error = hwcLayer->setColorTransform(getColorTransform());
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to setColorTransform: %s (%d)", mName.string(),
                to_string(error).c_str(), static_cast<int32_t>(error));
    }
    layerCompositionState.colorTransform = getColorTransform();

    error = hwcLayer->setSurfaceDamage(surfaceDamageRegion);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set surface damage: %s (%d)", mName.string(),
              to_string(error).c_str(), static_cast<int32_t>(error));
        surfaceDamageRegion.dump(LOG_TAG);
    }
    layerCompositionState.surfaceDamage = surfaceDamageRegion;
}
```

2. `BufferLayer::setPerFrameData`

BufferLayer的处理比ColorLayer多，Sideband，Cursor和其他的UI图层都属于BufferLayer，每种类型Layer处理都不同。

```cpp
//frameworks/native/services/surfaceflinger/BufferLayer.cpp
void BufferLayer::setPerFrameData(const sp<const DisplayDevice>& displayDevice,
                                  const ui::Transform& transform, const Rect& viewport,
                                  int32_t supportedPerFrameMetadata,
                                  const ui::Dataspace targetDataspace) {
    RETURN_IF_NO_HWC_LAYER(displayDevice);

    //在给HWC HAL层前，设置Display的投影的viewport给可见区域
    Region visible = transform.transform(visibleRegion.intersect(viewport));

    const auto outputLayer = findOutputLayerForDisplay(displayDevice);
    LOG_FATAL_IF(!outputLayer || !outputLayer->getState().hwc);

    auto& hwcLayer = (*outputLayer->getState().hwc).hwcLayer;
    //设置可见区域
    auto error = hwcLayer->setVisibleRegion(visible);
    outputLayer->editState().visibleRegion = visible;

    auto& layerCompositionState = getCompositionLayer()->editState().frontEnd;
    //设置Damage区域
    error = hwcLayer->setSurfaceDamage(surfaceDamageRegion);
    layerCompositionState.surfaceDamage = surfaceDamageRegion;

    // Sideband layers处理，默认为SIDEBAND合成
    if (layerCompositionState.sidebandStream.get()) {
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::SIDEBAND);
        ALOGV("[%s] Requesting Sideband composition", mName.string());
        error = hwcLayer->setSidebandStream(layerCompositionState.sidebandStream->handle());
        layerCompositionState.compositionType = Hwc2::IComposerClient::Composition::SIDEBAND;
        return;
    }
    // Device or Cursor layers
    //如果是Cursor Layer，则合成方式为CURSOR，其他为DEVICE合成
    if (mPotentialCursor) {
        ALOGV("[%s] Requesting Cursor composition", mName.string());
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::CURSOR);
    } else {
        ALOGV("[%s] Requesting Device composition", mName.string());
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::DEVICE);
    }

    ui::Dataspace dataspace = isColorSpaceAgnostic() && targetDataspace != ui::Dataspace::UNKNOWN
            ? targetDataspace
            : mCurrentDataSpace;
    //设置数据空间
    error = hwcLayer->setDataspace(dataspace);
    //HDR
    const HdrMetadata& metadata = getDrawingHdrMetadata();
    error = hwcLayer->setPerFrameMetadata(supportedPerFrameMetadata, metadata);
    //设置色彩矩阵
    error = hwcLayer->setColorTransform(getColorTransform());
    if (error == HWC2::Error::Unsupported) {
        //如果每个layer的色彩矩阵都不支持，则使用GPU合成（CLIENT）
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::CLIENT);
    } else if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to setColorTransform: %s (%d)", mName.string(),
                to_string(error).c_str(), static_cast<int32_t>(error));
    }
    layerCompositionState.dataspace = mCurrentDataSpace;
    layerCompositionState.colorTransform = getColorTransform();
    layerCompositionState.hdrMetadata = metadata;

    setHwcLayerBuffer(displayDevice);
}
```

***

## 2.4. beginFrame

```cpp
//SF.cpp
void SurfaceFlinger::beginFrame(const sp<DisplayDevice>& displayDevice) {
    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();

    bool dirty = !display->getDirtyRegion(false).isEmpty();
    bool empty = displayDevice->getVisibleLayersSortedByZ().size() == 0;
    bool wasEmpty = !displayState.lastCompositionHadVisibleLayers;
    //判断是否需要重新合成：
    //如果没有变化（即没有脏区域），不需要重新合成；
    //如果有一些变化，但是当前没有任何可见的Layers，并且最近一次的合成也没有，则跳过这次合成；
    //第二条判断做了以下两件事：
    //1.当所有layers从该Display被移除，我们将发射一个黑色（无内容）的帧，然后没有其他任何东西，直到我们获取到新的layers；
    //2.当一个Display被创建，包含了一个单独的空layer栈，我们将不会发射任何黑色的帧，知道一个Layers被添加到这个layer栈
    bool mustRecompose = dirty && !(empty && wasEmpty);
    ......
    //主显、外显：fsurfaceflinger/DisplayHardware/FramebufferSurface.h的beginFrame
    //虚拟显示：surfaceflinger/DisplayHardware/VirtualDisplaySurface.h的beginFrame
    display->getRenderSurface()->beginFrame(mustRecompose);

    if (mustRecompose) {
        display->editState().lastCompositionHadVisibleLayers = !empty;
    }
}
```

## 2.5. prepareFrame准备数据

```cpp
//SF.cpp
void SurfaceFlinger::prepareFrame(const sp<DisplayDevice>& displayDevice) {
    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();

    if (!displayState.isEnabled) {
        return;
    }

    status_t result = display->getRenderSurface()->prepareFrame();
    ALOGE_IF(result != NO_ERROR, "prepareFrame failed for %s: %d (%s)",
             displayDevice->getDebugName().c_str(), result, strerror(-result));
}
```

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngine/src/RenderSurface.cpp
status_t RenderSurface::prepareFrame() {
    auto& hwc = mCompositionEngine.getHwComposer();
    const auto id = mDisplay.getId();
    if (id) {
        //查看HWC是否支持之前SurfaceFlinger决定的合成方式
        status_t error = hwc.prepare(*id, mDisplay);
        if (error != NO_ERROR) {
            return error;
        }
    }
    //合成方式
    DisplaySurface::CompositionType compositionType;
    const bool hasClient = hwc.hasClientComposition(id);
    const bool hasDevice = hwc.hasDeviceComposition(id);
    if (hasClient && hasDevice) {
        compositionType = DisplaySurface::COMPOSITION_MIXED;
    } else if (hasClient) {
        compositionType = DisplaySurface::COMPOSITION_GLES;
    } else if (hasDevice) {
        compositionType = DisplaySurface::COMPOSITION_HWC;
    } else {
        // Nothing to do -- when turning the screen off we get a frame like
        // this. Call it a HWC frame since we won't be doing any GLES work but
        // will do a prepare/set cycle.
        compositionType = DisplaySurface::COMPOSITION_HWC;
    }
    return mDisplaySurface->prepareFrame(compositionType);
}
```

**重要函数：**

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
status_t HWComposer::prepare(DisplayId displayId, const compositionengine::Output& output) {
    ...
    displayData.validateWasSkipped = false;
    //SurfaceFlinger没有指定得有Client端合成，即false
    if (!displayData.hasClientComposition) {
        sp<Fence> outPresentFence;
        uint32_t state = UINT32_MAX;
        //调用函数尝试直接present，如果HWC不能直接显示，再执行validate操纵
        error = hwcDisplay->presentOrValidate(&numTypes, &numRequests, &outPresentFence , &state);
        if (error != HWC2::Error::HasChanges) {
            RETURN_IF_HWC_ERROR_FOR("presentOrValidate", error, displayId, UNKNOWN_ERROR);
        }
        //如果成功，数据显示，则不再执行后续流程
        if (state == 1) { //Present Succeeded.
            std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
            error = hwcDisplay->getReleaseFences(&releaseFences);
            displayData.releaseFences = std::move(releaseFences);
            displayData.lastPresentFence = outPresentFence;
            displayData.validateWasSkipped = true;
            displayData.presentError = error;
            return NO_ERROR;
        }
        // Present failed but Validate ran.
    } else {
        //调用validate
        error = hwcDisplay->validate(&numTypes, &numRequests);
    }
    ......
    std::unordered_map<HWC2::Layer*, HWC2::Composition> changedTypes;
    changedTypes.reserve(numTypes);
    //通过getChangedCompositionTypes函数获取到HWC对合成方式的修改，保存在changedTypes中
    error = hwcDisplay->getChangedCompositionTypes(&changedTypes);
    RETURN_IF_HWC_ERROR_FOR("getChangedCompositionTypes", error, displayId, BAD_INDEX);

    displayData.displayRequests = static_cast<HWC2::DisplayRequest>(0);
    std::unordered_map<HWC2::Layer*, HWC2::LayerRequest> layerRequests;
    layerRequests.reserve(numRequests);
    //获取LayerRequest，保存在layerRequests中
    error = hwcDisplay->getRequests(&displayData.displayRequests,
            &layerRequests);
    RETURN_IF_HWC_ERROR_FOR("getRequests", error, displayId, BAD_INDEX);
    .....
    //合成方式初始化为false
    displayData.hasClientComposition = false;
    displayData.hasDeviceComposition = false;
    //遍历Layer
    for (auto& outputLayer : output.getOutputLayersOrderedByZ()) {
        auto& state = outputLayer->editState();
        LOG_FATAL_IF(!state.hwc.);
        auto hwcLayer = (*state.hwc).hwcLayer;

        if (auto it = changedTypes.find(hwcLayer.get()); it != changedTypes.end()) {
            auto newCompositionType = it->second;
            validateChange(static_cast<HWC2::Composition>((*state.hwc).hwcCompositionType),
                           newCompositionType);
            (*state.hwc).hwcCompositionType =
                    static_cast<Hwc2::IComposerClient::Composition>(newCompositionType);
        }

        switch ((*state.hwc).hwcCompositionType) {
            case Hwc2::IComposerClient::Composition::CLIENT:
                displayData.hasClientComposition = true;
                break;
            case Hwc2::IComposerClient::Composition::DEVICE:
            case Hwc2::IComposerClient::Composition::SOLID_COLOR:
            case Hwc2::IComposerClient::Composition::CURSOR:
            case Hwc2::IComposerClient::Composition::SIDEBAND:
                displayData.hasDeviceComposition = true;
                break;
            default:
                break;
        }
        //响应layerRequests
        state.clearClientTarget = false;
        if (auto it = layerRequests.find(hwcLayer.get()); it != layerRequests.end()) {
            auto request = it->second;
            if (request == HWC2::LayerRequest::ClearClientTarget) {
                state.clearClientTarget = true;
            } else {
                LOG_DISPLAY_ERROR(displayId,
                                  ("Unknown layer request " + to_string(request)).c_str());
            }
        }
    }
    //最后，通过HWC，SurfaceFlinger接受修改
    error = hwcDisplay->acceptChanges();
    RETURN_IF_HWC_ERROR_FOR("acceptChanges", error, displayId, BAD_INDEX);

    return NO_ERROR;
}
```

**validate刷新：**

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
Error Display::validate(uint32_t* outNumTypes, uint32_t* outNumRequests)
{
    uint32_t numTypes = 0;
    uint32_t numRequests = 0;
    //调用Composer::validateDisplay
    auto intError = mComposer.validateDisplay(mId, &numTypes, &numRequests);
    auto error = static_cast<Error>(intError);
    if (error != Error::None && error != Error::HasChanges) {
        return error;
    }

    *outNumTypes = numTypes;
    *outNumRequests = numRequests;
    return error;
}
```

`validateDisplay`是通过CommandWriter写Buffer的方式调用到HWC中的，但是多了一个`execute`函数：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp
Error Composer::validateDisplay(Display display, uint32_t* outNumTypes,
        uint32_t* outNumRequests)
{
    mWriter.selectDisplay(display);
    //Buffer命令的调用，只是将命令写到Buffer中
    mWriter.validateDisplay();
    //真正的将触发HWC服务端解析Buffer命令，再分别取调HWC对应的实现函数
    Error error = execute();
    if (error != Error::NONE) {
        return error;
    }
    mReader.hasChanges(display, outNumTypes, outNumRequests);
    return Error::NONE;
}
```

## 2.6. doDebugFlashRegions

doDebugFlashRegions只是一个debug功能，受`mDebugRegion`控制（开发者选项）。

```cpp
void SurfaceFlinger::doDebugFlashRegions(const sp<DisplayDevice>& displayDevice,
                                         bool repaintEverything) {
    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();

    // is debugging enabled
    if (CC_LIKELY(!mDebugRegion))
        return;

    if (displayState.isEnabled) {
        // transform the dirty region into this screen's coordinate space
        const Region dirtyRegion = display->getDirtyRegion(repaintEverything);
        if (!dirtyRegion.isEmpty()) {
            base::unique_fd readyFence;
            // redraw the whole screen
            doComposeSurfaces(displayDevice, dirtyRegion, &readyFence);
            display->getRenderSurface()->queueBuffer(std::move(readyFence));
        }
    }

    postFramebuffer(displayDevice);

    if (mDebugRegion > 1) {
        usleep(mDebugRegion * 1000);
    }

    prepareFrame(displayDevice);
}
```

***

## 2.7. doComposition合成处理

```cpp
//SF.cpp
void SurfaceFlinger::doComposition(const sp<DisplayDevice>& displayDevice, bool repaintEverything) {
    ATRACE_CALL();
    ALOGV("doComposition");

    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();

    if (displayState.isEnabled) {
        //获取rebuildLayerStacks时计算的当前显示设备的脏区域DirtyRegion
        //如果是强制重画，mRepaintEverything为true，那么脏区域就是整个屏幕的大小
        //1. 获取脏区域
        const Region dirtyRegion = display->getDirtyRegion(repaintEverything);

        // repaint the framebuffer (if needed)
        //2. 主要是对当前的显示设备上的所有不支持硬件合成Layer进行OpenGL合成处理
        doDisplayComposition(displayDevice, dirtyRegion);

        display->editState().dirtyRegion.clear();
        display->getRenderSurface()->flip();
    }
    //统一交由HWC，由HWC硬件合成并输出到显示屏
    postFramebuffer(displayDevice);
}
```

***

**合成方式：**

+ Client合成
Client合成方式是相对与硬件合成来说的，其合成方式是，将各个Layer的内容用GPU渲染到暂存缓冲区中，最后将暂存缓冲区传送到显示硬件。这个暂存缓冲区，我们称为FBTarget，每个Display设备有各自的FBTarget。Client合成，之前称为GLES合成，我们也可以称之为GPU合成。Client合成，采用RenderEngine进行合成。

+ Device合成
就是用专门的硬件合成器进行合成HWComposer，所以硬件合成的能力就取决于硬件的实现。其合成方式是将各个Layer的数据全部传给显示硬件，并告知它从不同的缓冲区读取屏幕不同部分的数据。HWComposer是Devicehec的抽象。

***

### 2.7.1. getDirtyRegion获取脏区域

前面在`rebuildLayerStacks`重构Layer的时候，Display的脏区域DirtyRegion已经计算出来。如果重画，则mRepaintEverything为true，此时脏区域就是整个屏幕的大小。

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngine/src/Output.cpp
Region Output::getDirtyRegion(bool repaintEverything) const {
    Region dirty(mState.viewport);
    if (!repaintEverything) {
        dirty.andSelf(mState.dirtyRegion);
    }
    return dirty;
}
```

### 2.7.2. doDisplayComposition合成

合成方式主要就两种，一种Client客户端用GPU合成；另外一种，Device端HWC硬件合成。`doComposeSurfaces`主要是处理Client端合成，通过`RenderEngine`用GPU来进行合成。

```cpp
//SF.cpp
void SurfaceFlinger::doDisplayComposition(const sp<DisplayDevice>& displayDevice,
                                          const Region& inDirtyRegion) {
    auto display = displayDevice->getCompositionDisplay();
    // 只有在以下情况下才去真正合成这个Display的图像:
    // 1) 需要HWC硬件合成
    // 2) 脏区域不为空
    if (!displayDevice->getId() && inDirtyRegion.isEmpty()) {
        ALOGV("Skipping display composition");
        return;
    }

    ALOGV("doDisplayComposition");
    base::unique_fd readyFence;
    //调用doComposeSurfaces进行合成
    if (!doComposeSurfaces(displayDevice, Region::INVALID_REGION, &readyFence)) return;

    //将合成后的Buffer提交给该显示设备的BufferQueue,最终有FrameBufferSurface进行处理
    //swap buffers (presentation)
    //代码路径：frameworks/native/libs/gui/Surface.cpp
    display->getRenderSurface()->queueBuffer(std::move(readyFence));
}
```

#### 2.7.2.1. *doComposeSurfaces

```cpp
//SF.cpp
//doComposeSurfaces
bool SurfaceFlinger::doComposeSurfaces(const sp<DisplayDevice>& displayDevice,
                                       const Region& debugRegion, base::unique_fd* readyFence) {
    .....
    //RenderEngine初始化
    const Region bounds(displayState.bounds);
    const DisplayRenderArea renderArea(displayDevice);
    const bool hasClientComposition = getHwComposer().hasClientComposition(displayId);
    ATRACE_INT("hasClientComposition", hasClientComposition);

    bool applyColorMatrix = false;
    renderengine::DisplaySettings clientCompositionDisplay;
    std::vector<renderengine::LayerSettings> clientCompositionLayers;
    sp<GraphicBuffer> buf;
    base::unique_fd fd;

    if (hasClientComposition) {
        ALOGV("hasClientComposition");

        if (displayDevice->isPrimary() && supportProtectedContent) {
            bool needsProtected = false;
            //遍历layer
            for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
                //如果layer受保护，则进行标记
                if (layer->isProtected()) {
                    needsProtected = true;
                    break;
                }
            }
            if (needsProtected != renderEngine.isProtected()) {
                renderEngine.useProtectedContext(needsProtected);
            }
            if (needsProtected != display->getRenderSurface()->isProtected() &&
                needsProtected == renderEngine.isProtected()) {
                display->getRenderSurface()->setProtected(needsProtected);
            }
        }
        ...
        //Client合成Display的属性赋值
        clientCompositionDisplay.physicalDisplay = displayState.scissor; //主屏剪切区
        clientCompositionDisplay.clip = displayState.scissor;
        const ui::Transform& displayTransform = displayState.transform;
        clientCompositionDisplay.globalTransform = displayTransform.asMatrix4();
        clientCompositionDisplay.orientation = displayState.orientation;  //屏幕转向

        const auto* profile = display->getDisplayColorProfile();
        Dataspace outputDataspace = Dataspace::UNKNOWN;
        //是否用WideColor，设置数据空间
        if (profile->hasWideColorGamut()) {
            outputDataspace = displayState.dataspace;
        }
        clientCompositionDisplay.outputDataspace = outputDataspace;
        clientCompositionDisplay.maxLuminance =
        ...
        if (applyColorMatrix) {
            clientCompositionDisplay.colorTransform = displayState.colorTransformMat;
        }
    }
    ......

    //将Client端Layer渲染到FrameBuffer（即FBTarget）
    ALOGV("Rendering client layers");
    bool firstLayer = true;
    Region clearRegion = Region::INVALID_REGION;
    //遍历layer
    for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
        const Region viewportRegion(displayState.viewport);
        const Region clip(viewportRegion.intersect(layer->visibleRegion));
        if (!clip.isEmpty()) {
            switch (layer->getCompositionType(displayDevice)) {
                case Hwc2::IComposerClient::Composition::CURSOR:
                case Hwc2::IComposerClient::Composition::DEVICE:
                case Hwc2::IComposerClient::Composition::SIDEBAND:
                case Hwc2::IComposerClient::Composition::SOLID_COLOR: {
                    LOG_ALWAYS_FATAL_IF(!displayId);
                    const Layer::State& state(layer->getDrawingState());
                    if (layer->getClearClientTarget(displayDevice) && !firstLayer &&
                        layer->isOpaque(state) && (layer->getAlpha() == 1.0f) &&
                        layer->getRoundedCornerState().radius == 0.0f && hasClientComposition) {
                        //千万不要清除第一层，因为我们保证FB已经清除了
                        renderengine::LayerSettings layerSettings;
                        Region dummyRegion;
                        //调用该函数，此处有对受保护secure layer的处理（如果有数字保护协议，则会显示black layer）
                        //然后会将layer的clip区域绘制到FBTarget上
                        bool prepared =
                                layer->prepareClientLayer(renderArea, clip, dummyRegion,
                                                          supportProtectedContent, layerSettings);

                        if (prepared) {
                            layerSettings.source.buffer.buffer = nullptr;
                            layerSettings.source.solidColor = half3(0.0, 0.0, 0.0);
                            layerSettings.alpha = half(0.0);
                            layerSettings.disableBlending = true;
                            clientCompositionLayers.push_back(layerSettings);
                        }
                    }
                    break;
                }
                case Hwc2::IComposerClient::Composition::CLIENT: {
                    renderengine::LayerSettings layerSettings;
                    //同上
                    bool prepared =
                            layer->prepareClientLayer(renderArea, clip, clearRegion,
                                                      supportProtectedContent, layerSettings);
                    if (prepared) {
                        clientCompositionLayers.push_back(layerSettings);
                    }
                    break;
                }
                default:
                    break;
            }
        } else {
            ALOGV("  Skipping for empty clip");
        }
        firstLayer = false;
    }
    ......
}
}
```

### 2.7.3. postFramebuffer

```cpp
//SF.cpp
void SurfaceFlinger::postFramebuffer(const sp<DisplayDevice>& displayDevice) {
    ATRACE_CALL();
    ALOGV("postFramebuffer");

    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();
    const auto displayId = display->getId();

    if (displayState.isEnabled) {
        if (displayId) {
            //调用该函数获取releaseFence
            getHwComposer().presentAndGetReleaseFences(*displayId);
        }
        display->getRenderSurface()->onPresentDisplayCompleted();
        for (auto& layer : display->getOutputLayersOrderedByZ()) {
            sp<Fence> releaseFence = Fence::NO_FENCE;
            bool usedClientComposition = true;

            //在这一帧的fence发出信号的时候，上一帧的layer buffer被HWC释放
            //始终先从HWC处获取释放fence
            if (layer->getState().hwc) {
                const auto& hwcState = *layer->getState().hwc;
                releaseFence =
                        getHwComposer().getLayerReleaseFence(*displayId, hwcState.hwcLayer.get());
                usedClientComposition =
                        hwcState.hwcCompositionType == Hwc2::IComposerClient::Composition::CLIENT;
            }

            //如果层在上一帧中是Client客户端合成的，则需要与前一个Client客户端获取的fence合并。
            //因为我们不跟踪它，所以当它可用时，总是与当前客户端Fence合并，即使这是子选项
            if (usedClientComposition) {
                releaseFence =
                        Fence::merge("LayerRelease", releaseFence,
                                     display->getRenderSurface()->getClientTargetAcquireFence());
            }

            layer->getLayerFE().onLayerDisplayed(releaseFence);
        }

        //我们有一列需要fence的layer，它们与display->getVisibleLayersSortedByZ不相交。
        //我们能做的最好的就是给他们提供现有的fence
        if (!displayDevice->getLayersNeedingFences().isEmpty()) {
            sp<Fence> presentFence =
                    displayId ? getHwComposer().getPresentFence(*displayId) : Fence::NO_FENCE;
            for (auto& layer : displayDevice->getLayersNeedingFences()) {
                layer->getCompositionLayer()->getLayerFE()->onLayerDisplayed(presentFence);
            }
        }

        if (displayId) {
            //清除
            getHwComposer().clearReleaseFences(*displayId);
        }
    }
}

```

***

## 2.8. postComposition

```cpp
//SF.cpp
void SurfaceFlinger::postComposition()
{
    ATRACE_CALL();
    ALOGV("postComposition");

    //调用Layer的onPostComposition， 处理Layer中刚刚绘制的Buffer的Fence
    //此时才会真正释放掉一个Buffer
    nsecs_t dequeueReadyTime = systemTime();
    for (auto& layer : mLayersWithQueuedFrames) {
        //这一帧合成完成后，将会被替代的Buffer释放掉
        layer->releasePendingBuffer(dequeueReadyTime);
    }
    ...
    //记录Buffer状态
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        bool frameLatched =
                layer->onPostComposition(displayDevice->getId(), glCompositionDoneFenceTime,
                                         presentFenceTime, compositorTiming);
        if (frameLatched) {
            recordBufferingStats(layer->getName().string(),
                    layer->getOccupancyHistory(false));
        }
    }
    //VSYNC是由mScheduler分发出来的，并不是每一次都是从底层硬件上报的
    //所以mScheduler需要和底层硬件Vsync保持同步
    if (presentFenceTime->isValid()) {
        mScheduler->addPresentFence(presentFenceTime);
    }
    //Vsync同步
    if (!hasSyncFramework) {
        if (displayDevice && getHwComposer().isConnected(*displayDevice->getId()) &&
            displayDevice->isPoweredOn()) {
            mScheduler->enableHardwareVsync();
        }
    }
    //动画合成处理
    if (mAnimCompositionPending) {
        mAnimCompositionPending = false;

        if (presentFenceTime->isValid()) {
            mAnimFrameTracker.setActualPresentFence(
                    std::move(presentFenceTime));
        } else if (displayDevice && getHwComposer().isConnected(*displayDevice->getId())) {
            // The HWC doesn't support present fences, so use the refresh
            // timestamp instead.
            const nsecs_t presentTime =
                    getHwComposer().getRefreshTimestamp(*displayDevice->getId());
            mAnimFrameTracker.setActualPresentTime(presentTime);
        }
        //advanceFrame处理FBTarget
        mAnimFrameTracker.advanceFrame();
    }
    ......
    //处理时间的记录
    }
```

至此，REFRESH处理完成。之后等到下一个Vsync周期，开始下一次合成。

### 2.8.1. advanceFrame

其中调用advanceFrame方法，虚显用的`VirtualDisplaySurface`，非虚显用的`FramebufferSurface`。advanceFrame获取FBTarget的数据：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::advanceFrame() {
    uint32_t slot = 0;
    sp<GraphicBuffer> buf;
    sp<Fence> acquireFence(Fence::NO_FENCE);
    Dataspace dataspace = Dataspace::UNKNOWN;
    status_t result = nextBuffer(slot, buf, acquireFence, dataspace);
    mDataSpace = dataspace;
    if (result != NO_ERROR) {
        ALOGE("error latching next FramebufferSurface buffer: %s (%d)",
                strerror(-result), result);
    }
    return result;
}

status_t FramebufferSurface::nextBuffer(uint32_t& outSlot,
        sp<GraphicBuffer>& outBuffer, sp<Fence>& outFence,
        Dataspace& outDataspace) {
    Mutex::Autolock lock(mMutex);
    BufferItem item;
    //1.获取一个Buffer（在上面doComposition - doDisplayComposition函数最后调用queueBuffer，会放到FrameBufferSurface的BufferQueue中）
    //此处的函数将从这个BufferQueue中获取一个Buffer
    status_t err = acquireBufferLocked(&item, 0);
    ...
    //2.当前Buffer序号mCurrentBufferSlot，当前Buffer是mCurrentBuffer，对应的Fence是mCurrentFence
    //如果上面获取到的Buffer不一样，则将Current置为Previous上一个buffer，否则没有变化
    if (mCurrentBufferSlot != BufferQueue::INVALID_BUFFER_SLOT &&
        item.mSlot != mCurrentBufferSlot) {
        mHasPendingRelease = true;
        mPreviousBufferSlot = mCurrentBufferSlot;
        mPreviousBuffer = mCurrentBuffer;
    }
    //3.将获取到的Buffer置为当前的Current
    mCurrentBufferSlot = item.mSlot;
    mCurrentBuffer = mSlots[mCurrentBufferSlot].mGraphicBuffer;
    mCurrentFence = item.mFence;

    outFence = item.mFence;
    mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer, &outSlot, &outBuffer);
    outDataspace = static_cast<Dataspace>(item.mDataSpace);
    //4.将FBTarget设置给HWC  （头文件HWComposer& mHwc;）
    status_t result = mHwc.setClientTarget(mDisplayId, outSlot, outFence, outBuffer, outDataspace);
    if (result != NO_ERROR) {
        ALOGE("error posting framebuffer: %d", result);
        return result;
    }
    return NO_ERROR;
}
```

setClientTarget函数：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
status_t HWComposer::setClientTarget(DisplayId displayId, uint32_t slot,
                                     const sp<Fence>& acquireFence, const sp<GraphicBuffer>& target,
                                     ui::Dataspace dataspace) {
    //头文件：std::unordered_map<DisplayId, DisplayData> mDisplayData;
    auto& hwcDisplay = mDisplayData[displayId].hwcDisplay;
    //FBTarget是通过Command Buffer的方式传到HWC中的
    auto error = hwcDisplay->setClientTarget(slot, target, acquireFence, dataspace);
    return NO_ERROR;
}

```

HWComposer头文件，hwcDisplay在HWC2命名空间内。

```cpp
    struct DisplayData {
        bool isVirtual = false;
        bool hasClientComposition = false;
        bool hasDeviceComposition = false;
        HWC2::Display* hwcDisplay = nullptr;
        HWC2::DisplayRequest displayRequests;
        sp<Fence> lastPresentFence = Fence::NO_FENCE; // signals when the last set op retires
        std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
        buffer_handle_t outbufHandle = nullptr;
        sp<Fence> outbufAcquireFence = Fence::NO_FENCE;
        mutable std::unordered_map<int32_t,
                std::shared_ptr<const HWC2::Display::Config>> configMap;

        bool validateWasSkipped;
        HWC2::Error presentError;

        bool vsyncTraceToggle = false;

        std::mutex vsyncEnabledLock;
        HWC2::Vsync vsyncEnabled GUARDED_BY(vsyncEnabledLock) = HWC2::Vsync::Disable;

        mutable std::mutex lastHwVsyncLock;
        nsecs_t lastHwVsync GUARDED_BY(lastHwVsyncLock) = 0;
    };
```

***

# 3. GPU合成模块概述

硬件HWC合成是由Vendor实现。而各个厂商在这部分的实现不同。

GPU合成（Client）是Android原生自带的，本质是采用GPU进程合成，SurfaceFlinger模块封装了RenderEngine进行具体的实现。

看一下这个模块的文件目录：

```s
//frameworks/native/libs/renderengine
├── Android.bp
├── Description.cpp
├── gl
│   ├── GLESRenderEngine.cpp
│   ├── GLESRenderEngine.h
│   ├── GLExtensions.cpp
│   ├── GLExtensions.h
│   ├── GLFramebuffer.cpp
│   ├── GLFramebuffer.h
│   ├── GLImage.cpp
│   ├── GLImage.h
│   ├── ProgramCache.cpp
│   ├── ProgramCache.h
│   ├── Program.cpp
│   └── Program.h
├── include
│   └── renderengine
│       ├── DisplaySettings.h
│       ├── Framebuffer.h
│       ├── Image.h
│       ├── LayerSettings.h
│       ├── Mesh.h
│       ├── mock
│       │   ├── Framebuffer.h
│       │   ├── Image.h
│       │   └── RenderEngine.h
│       ├── private
│       │   └── Description.h
│       ├── RenderEngine.h
│       └── Texture.h
├── Mesh.cpp
├── mock
│   ├── Framebuffer.cpp
│   ├── Image.cpp
│   └── RenderEngine.cpp
├── OWNERS
├── RenderEngine.cpp
├── TEST_MAPPING
├── tests
│   ├── Android.bp
│   └── RenderEngineTest.cpp
└── Texture.cpp
```

***

## 3.1. 创建RenderEngine

在SF.cpp初始化函数init中：

```cpp
void SurfaceFlinger::init() {
    ...
    // TODO(b/77156734): We need to stop casting and use HAL types when possible.
    // Sending maxFrameBufferAcquiredBuffers as the cache size is tightly tuned to single-display.
    mCompositionEngine->setRenderEngine(
            renderengine::RenderEngine::create(static_cast<int32_t>(defaultCompositionPixelFormat),
                                               renderEngineFeature, maxFrameBufferAcquiredBuffers));
    ...
}
```

create调用RenderEngine.cpp中的对应函数：（在Q版本该模块已经独立出来，该模块是对GPU渲染的封装）

```cpp
//frameworks/native/libs/renderengine/RenderEngine.cpp
std::unique_ptr<impl::RenderEngine> RenderEngine::create(int hwcFormat, uint32_t featureFlags,
                                                         uint32_t imageCacheSize) {
    char prop[PROPERTY_VALUE_MAX];
    property_get(PROPERTY_DEBUG_RENDERENGINE_BACKEND, prop, "gles");
    if (strcmp(prop, "gles") == 0) {
        ALOGD("RenderEngine GLES Backend");
        return renderengine::gl::GLESRenderEngine::create(hwcFormat, featureFlags, imageCacheSize);
    }
    ALOGE("UNKNOWN BackendType: %s, create GLES RenderEngine.", prop);
    return renderengine::gl::GLESRenderEngine::create(hwcFormat, featureFlags, imageCacheSize);
}
```

```cpp
//frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp
std::unique_ptr<GLESRenderEngine> GLESRenderEngine::create(int hwcFormat, uint32_t featureFlags,
                                                           uint32_t imageCacheSize) {
    //创建EGLDisplay
    EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    //初始化EGLDisplay
    if (!eglInitialize(display, nullptr, nullptr)) {
        LOG_ALWAYS_FATAL("failed to initialize EGL");
    }
    //选择EGLConfig
    EGLConfig config = EGL_NO_CONFIG;
    if (!extensions.hasNoConfigContext()) {
        config = chooseEglConfig(display, hwcFormat, /*logConfig*/ true);
    }
    ...
    //创建EGLContext
    //在该函数中:
    //1.调用eglGetConfigAttrib获取renderableType、
    //2.初始化Context属性contextAttributes、
    //3.调用eglCreateContext创建EGLContext
    if ((featureFlags & RenderEngine::ENABLE_PROTECTED_CONTEXT) &&
        extensions.hasProtectedContent()) {
        protectedContext = createEglContext(display, config, nullptr, useContextPriority,
                                            Protection::PROTECTED);
        ALOGE_IF(protectedContext == EGL_NO_CONTEXT, "Can't create protected context");
    }

    EGLContext ctxt = createEglContext(display, config, protectedContext, useContextPriority,
                                       Protection::UNPROTECTED);
    ...
    EGLSurface dummy = EGL_NO_SURFACE;
    if (!extensions.hasSurfacelessContext()) {
        //创建PBuffer
        dummy = createDummyEglPbufferSurface(display, config, hwcFormat, Protection::UNPROTECTED);
    }
    ...
    //查看可以获取到什么版本的GL
    GlesVersion version = parseGlesVersion(extensions.getVersion());

    //初始化当前GL的渲染器RenderEngine
    std::unique_ptr<GLESRenderEngine> engine;
    switch (version) {
        case GLES_VERSION_1_0:
        case GLES_VERSION_1_1:
            LOG_ALWAYS_FATAL("SurfaceFlinger requires OpenGL ES 2.0 minimum to run.");
            break;
        case GLES_VERSION_2_0:
        case GLES_VERSION_3_0:
            engine = std::make_unique<GLESRenderEngine>(featureFlags, display, config, ctxt, dummy,
                                                        protectedContext, protectedDummy,
                                                        imageCacheSize);
            break;
    }
    ......
}
```

## 3.2. 创建Surface FBTarget

RenderEngine创建时始化的EGLDisplaym，EGLConfig，EGLContext等，都是所有Display共用的。

而Surface每个Display的是自己的，在创建DisplayDevice时，创建对应的Surface。

从BufferQueue中dequeue Buffer进行渲染，swapBuffer时，也queue到Bufferqueu中。这里的ANativeWindow，本质就是FBTarget。

## 3.3. 创建Texture纹理

在BufferLayer创建的构造函数中创建Texture：

```cpp
//BufferLayer.cpp
BufferLayer::BufferLayer(const LayerCreationArgs& args)
      : Layer(args),
      //调用SurfaceFlinger的getNewTexture创建
      //在创建BufferLayerConsumer时，传到了Consumer中，对应的值为mTexName
        mTextureName(args.flinger->getNewTexture()),
        mCompositionLayer{mFlinger->getCompositionEngine().createLayer(
                compositionengine::LayerCreationArgs{this})} {
                    ......
}
```

```cpp
//SF.cpp
uint32_t SurfaceFlinger::getNewTexture() {
    ......
    // The pool was empty, so we need to get a new texture name directly using a
    // blocking call to the main thread
    uint32_t name = 0;
    postMessageSync(new LambdaMessage([&]() { getRenderEngine().genTextures(1, &name); }));
    return name;
}
```

调用genTextures：

```cpp
//frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp
void GLESRenderEngine::genTextures(size_t count, uint32_t* names) {
    glGenTextures(count, names);  //生成Texture，在BufferLayer中保存在mTexture中
}
```

## 3.4. 绑定Texture纹理

在`BufferLayerConsumer::updateTexImage`函数中调用`bindTextureImageLocked`绑定新的buffer到GL Texture纹理。

而该函数是在SurfaceFlinger调用latchBuffer从BufferQueue申请获取渲染好的buffer的时候会调用到。

## 3.5. Layer合成

在Q版本之前是调用的`onDraw`函数，而在Q上函数名变成`prepareClientLayer`。

```cpp
bool BufferLayer::prepareClientLayer(const RenderArea& renderArea, const Region& clip,
                                     bool useIdentityTransform, Region& clearRegion,
                                     const bool supportProtectedContent,
                                     renderengine::LayerSettings& layer) {
                                        ...
    //DRM处理，是否阻塞当前Layer                                        
    bool blackOutLayer =
            (isProtected() && !supportProtectedContent) || (isSecure() && !renderArea.isSecure());
    ...
}
```

在`SurfaceFlinger::doComposeSurfaces`函数中，调用完prepareClientLayer后，末尾最后调用`renderEngine.drawLayers`函数。

最终调用到GPU合成模块`frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp`的`GLESRenderEngine::drawLayers`函数。

```cpp
status_t GLESRenderEngine::drawLayers(const DisplaySettings& display,
                                      const std::vector<LayerSettings>& layers,
                                      ANativeWindowBuffer* const buffer,
                                      const bool useFramebufferCache, base::unique_fd&& bufferFence,
                                      base::unique_fd* drawFence) {
            ...
            //Texture坐标顶点
            renderengine::Mesh::VertexArray<vec2> texCoords(mesh.getTexCoordArray<vec2>());
            texCoords[0] = vec2(0.0, 0.0);
            texCoords[1] = vec2(0.0, 1.0);
            texCoords[2] = vec2(1.0, 1.0);
            texCoords[3] = vec2(1.0, 0.0);
            ...
            // Buffer sources will have a black solid color ignored in the shader,
            // so in that scenario the solid color passed here is arbitrary.
            //处理Alpha的Blend
            setupLayerBlending(usePremultipliedAlpha, isOpaque, disableTexture, color,
                               layer.geometry.roundedCornersRadius);
                                      }
            ...
            // We only want to do a special handling for rounded corners when having rounded corners
            // is the only reason it needs to turn on blending, otherwise, we handle it like the
            // usual way since it needs to turn on blending anyway.
            if (layer.geometry.roundedCornersRadius > 0.0 && color.a >= 1.0f && isOpaque) {
                handleRoundedCorners(display, layer, mesh);
            } else {
                //绘制（合成）内容
                //该函数中使用glDrawArrays函数进行绘制（合成）
                drawMesh(mesh);
            }
```

***

# 4. 参考文章

+ [Google Developers - BlendMode](https://developer.android.google.cn/reference/android/graphics/BlendMode)
+ [SurfaceFlinger合成流程(一)](https://www.jianshu.com/p/fa115146949f)
+ [SurfaceFlinger合成流程(二)](https://www.jianshu.com/p/fd16dcb4dfb6)
+ [Android Handler消息循环处理机制](https://wizzie.top/Blog/2019/09/22/2019/190922-android-handler-cpp/#SurfaceFlinger%E7%9A%84%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6)
+ [Android 图形显示框架](https://wizzie.top/Blog/2020/07/30/2020/200730_android_GraphicsFramework/)
+ [Android BitTube](https://blog.csdn.net/u013686019/article/details/51614774)
+ [Android之BitTube](https://blog.csdn.net/dabenxiong666/article/details/80629316)
+ [基于Android Q分析SurfaceFlinger启动过程](https://blog.csdn.net/weixin_41054077/article/details/105735639)
+ [Android SurfaceFlinger和HWC2概述 - mCurrentState和mDrawingState](https://wizzie.top/Blog/2019/12/22/2019/191222_android_HWC2/#mCurrentState%E5%92%8CmDrawingState)
+ [SurfaceFlinger图像合成[1]](https://www.jianshu.com/p/b0928eaaeb1c)