---
layout: single
related: false
title:  Android InputDispatcher获取点击事件
date:   2020-05-20 23:52:00
categories: android
tags: input display
toc: true
---

> Input点击事件从InputReader会传到InputDispatcher进行处理。针对一些点击调试方式和日志打印，来分析InputDispatcher获取点击事件的部分流程。  
> 参考上一篇《Android 触控事件分析》

<!--mode-->

# 1. Input down/up事件查看

在开发者选项打开“显示点按操作反馈”和“指针位置”，通过`adb shell getevent -lrt`命令，然后点击屏幕可以查看到控制台的打印。

查看帮助：`adb shell getevent -h`

打印结果包含Input的down/up事件，以及点击点的坐标（十六进制）:

```log
[    1423.973137] /dev/input/event2: EV_ABS       ABS_MT_TRACKING_ID   0000003b            
[    1423.973137] /dev/input/event2: EV_ABS       ABS_MT_POSITION_X    0000017e            //横坐标X=382  十六进制转成十进制=》 1*16*16+7*16+14*1=382
[    1423.973137] /dev/input/event2: EV_ABS       ABS_MT_POSITION_Y    0000032d            //纵坐标Y=813  十六进制转成十进制=》 3*16*16+2*16+13*1=813  
[    1423.973137] /dev/input/event2: EV_ABS       ABS_MT_TOUCH_MAJOR   0000000a            
[    1423.973137] /dev/input/event2: EV_ABS       ABS_MT_PRESSURE      000003e8            
[    1423.973137] /dev/input/event2: EV_KEY       BTN_TOUCH            DOWN                
[    1423.973137] /dev/input/event2: EV_SYN       SYN_REPORT           00000000            
[    1436.084174] /dev/input/event2: EV_ABS       ABS_MT_TOUCH_MAJOR   00000000            
[    1436.084174] /dev/input/event2: EV_ABS       ABS_MT_PRESSURE      00000000            
[    1436.084174] /dev/input/event2: EV_ABS       ABS_MT_TRACKING_ID   ffffffff            
[    1436.084174] /dev/input/event2: EV_KEY       BTN_TOUCH            UP                  
[    1436.084174] /dev/input/event2: EV_SYN       SYN_REPORT           00000000             rate 0
```

# 2. systrace分析

抓取Systrace可以查看到触屏的整个事件，从InputReader开始，然后到deliverInputEvent触发APP绘制。关于报点可以重点关注inputflinger模块的log打印，会打印input的坐标。
(参考：http://wizzie.top/2020/03/17/2020/200317_adnroid_touchEvent/)

# 3. input debug开关打开抓取日志分析

```log
adb root
adb shell setprop sys.input.TouchFilterEnable true
adb shell setprop sys.input.TouchFilterLogEnable true
adb shell dumpsys window -d enable DEBUG_FOCUS
adb shell dumpsys window -d enable DEBUG_INPUT
adb shell setprop sys.inputlog.enabled true
adb shell dumpsys input
```

然后抓取log可以看到类似`InputDispatcher: notifyMotion`、`dispatchMotion`这些日志打印。

# 4. 日志打印分析代码流程

## 4.1. inputReader通过QueuedInputListener

负责读取触摸事件交给 InputDispatcher 进行事件派发。

1. 首先在构造函数中new一个QueueListener对象：

```cpp
//frameworks/native/services/inputflinger/InputReader.cpp
InputReader::InputReader(const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& policy,
        const sp<InputListenerInterface>& listener) :
        mContext(this), mEventHub(eventHub), mPolicy(policy),
        mNextSequenceNum(1), mGlobalMetaState(0), mGeneration(1),
        mDisableVirtualKeysTimeout(LLONG_MIN), mNextTimeout(LLONG_MAX),
        mConfigurationChangesToRefresh(0) {
    /// M: for nwk @{
    void *func;
    /// @}
    mQueuedListener = new QueuedInputListener(listener);
    ......
```

2. 在`InputReader::loopOnce()`循环等待消息

```cpp
void InputReader::loopOnce() {
    int32_t oldGeneration;
    int32_t timeoutMillis;
    bool inputDevicesChanged = false;
    std::vector<InputDeviceInfo> inputDevices;
    ...
    mQueuedListener->flush();
}
```

3. flush刷新将遍历QueuedInputListener中`mArgsQueue`的数组元素，触发每一个元素NotifyArgs的`notify`方法，交给内部InputDispatcher，清空数组。

```cpp
//frameworks/native/services/inputflinger/InputListener.cpp
void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}
```

4. 结构体NotifyMotionArgs/NotifySwitchArgs/NotifyDeviceResetArgs继承自NotifyArgs，所以执行NotifyArgs的`notify`函数。调用派发者InputDispatcher的通知notifyMotion，将自己交给派发者。

```cpp
//frameworks/native/services/inputflinger/InputListener.cpp
void NotifyMotionArgs::notify(const sp<InputListenerInterface>& listener) const {
    listener->notifyMotion(this);
}
```

## 4.2. InputDispatcher获取数据

1. 触发InputDispatcher.cpp的`notifyMotion`函数，读取线程InputReaderThread在处理事务，notifyMotion方法之后会唤醒分发线程，接下来的任务就由分发线程处理。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::notifyMotion(const NotifyMotionArgs* args) {
#if DEBUG_INBOUND_EVENT_DETAILS   //打开了input debug log后会打印以下log
    ALOGD("notifyMotion - eventTime=%" PRId64 ", deviceId=%d, source=0x%x, displayId=%" PRId32
            ", policyFlags=0x%x, "
            "action=0x%x, actionButton=0x%x, flags=0x%x, metaState=0x%x, buttonState=0x%x,"
            "edgeFlags=0x%x, xPrecision=%f, yPrecision=%f, downTime=%" PRId64,
            args->eventTime, args->deviceId, args->source, args->displayId, args->policyFlags,
            args->action, args->actionButton, args->flags, args->metaState, args->buttonState,
            args->edgeFlags, args->xPrecision, args->yPrecision, args->downTime);
    for (uint32_t i = 0; i < args->pointerCount; i++) {
        ALOGD("  Pointer %d: id=%d, toolType=%d, "
                "x=%f, y=%f, pressure=%f, size=%f, "
                "touchMajor=%f, touchMinor=%f, toolMajor=%f, toolMinor=%f, "
                "orientation=%f",
                i, args->pointerProperties[i].id,
                args->pointerProperties[i].toolType,
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_X),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_Y),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_PRESSURE),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_SIZE),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_TOUCH_MAJOR),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_TOUCH_MINOR),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_TOOL_MAJOR),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_TOOL_MINOR),
                args->pointerCoords[i].getAxisValue(AMOTION_EVENT_AXIS_ORIENTATION));
    }
#endif
......
        // Just enqueue a new motion event. //将NotifyMotionArgs的数据封装为MotionEntry
        MotionEntry* newEntry = new MotionEntry(args->sequenceNum, args->eventTime,
                args->deviceId, args->source, args->displayId, policyFlags,
                args->action, args->actionButton, args->flags,
                args->metaState, args->buttonState, args->classification,
                args->edgeFlags, args->xPrecision, args->yPrecision, args->downTime,
                args->pointerCount, args->pointerProperties, args->pointerCoords, 0, 0);
        //插入InputDispatcher的mInboundQueue队列中
        needWake = enqueueInboundEventLocked(newEntry);
        mLock.unlock();
    } // release lock

    if (needWake) {   //需要唤醒分发线程
        mLooper->wake();  //TODO
    }
}
```

**Note：**注意：mLooper属于InputDispatcher，InputManager创建InputDispatcher时，在其构造方法同时创建mLooper，创建的线程是服务线程，并非读取或分发线程
这里只是借用了Looper提供的epoll唤醒与休眠机制，在分发线程中InputDispatcherThread中使用mLooper休眠，读取线程负责唤醒。

2. 数据封装成MotionEntry，然后作为enqueueInboundEventLocked函数的入参，插入到mInboundQueue队列尾部。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcher::enqueueInboundEventLocked(EventEntry* entry) {
    bool needWake = mInboundQueue.isEmpty();
    mInboundQueue.enqueueAtTail(entry);
    traceInboundQueueLengthLocked();
```

## 4.3. InputDispatcherThread分发线程被唤醒

> 参考：http://wizzie.top/2020/05/10/2020/200510_android_inputANR/

1. 在InputDispatcherThread线程threadLoop循环中，触发InputDispatcher的dispatchOnce方法。然后调用dispatchOnce方法。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}

void InputDispatcher::dispatchOnce() {
    //下次唤醒事件，设置无限大
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
        std::scoped_lock _l(mLock);
        mDispatcherIsAlive.notify_all();

        //mCommandQueue为空时，触发dispatchOnceInnerLocked
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }

        // Run all pending commands if there are any.
        // If any commands were run then force the next poll to wake up immediately.
        if (runCommandsLockedInterruptible()) {         //mCommandQueue为空时是false
            nextWakeupTime = LONG_LONG_MIN;
        }
    } // release lock

    // Wait for callback or timeout or wake.  (make sure we round up, not down)
    nsecs_t currentTime = now();
    //计算下一次唤醒时间，比当前时间大
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    mLooper->pollOnce(timeoutMillis);
}
```

2. 利用Looper在epoll_wait处进入休眠，休眠timeoutMillis时间仍无事件，threadLoop会一直循环，继续dispatchOnce。
当被唤醒时，执行switch循环进入dispatchOnceInnerLocked取出队列中的事件。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ...
        // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (! mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            ...
        } else {
            // Inbound queue has at least one entry.
            mPendingEvent = mInboundQueue.dequeueAtHead();
            traceInboundQueueLengthLocked();
        }

        // Poke user activity for this event.
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            pokeUserActivityLocked(mPendingEvent);
        }

        // Get ready to dispatch the event.
        resetANRTimeoutsLocked();
    }
    ...

    //mPendingEvent的type做区分处理，此处对motion事件分析
    ALOG_ASSERT(mPendingEvent != nullptr);
    ...
    switch (mPendingEvent->type) {
        ...
        case EventEntry::TYPE_MOTION: {
        MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
        //如果没有及时响应窗口切换操作
        if (dropReason == DROP_REASON_NOT_DROPPED && isAppSwitchDue) {
            dropReason = DROP_REASON_APP_SWITCH;
        }
        //事件过期
        if (dropReason == DROP_REASON_NOT_DROPPED
                && isStaleEvent(currentTime, typedEntry)) {
            dropReason = DROP_REASON_STALE;
        }
        //阻碍其他窗口获取事件
        if (dropReason == DROP_REASON_NOT_DROPPED && mNextUnblockedEvent) {
            dropReason = DROP_REASON_BLOCKED;
        }
        //此处执行事件
        done = dispatchMotionLocked(currentTime, typedEntry,
                &dropReason, nextWakeupTime);
        break;
    }
            ...

```

**Notes:**Looper借助epoll机制实现线程休眠，它本身内部有套接字mWakeEventFd，在rebuildEpollLocked建立时，注册到epoll_ctl监听。因此wake方法就是向mWakeEventFd套接字发送一段字符，促使epoll_wait处的线程能监听到，从而InputDispatcherThread线程被唤醒。

## 4.4. InputDispatcher事件处理

1. InputDispatcher#dispatchMotionLocked处理MotionEntry。此处函数开头会有类似`InputDispatcher: dispatchMotion - eventTime= ...`的日志打印。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
//入参：dropReason代表了事件丢弃的原因，它的默认值为DROP_REASON_NOT_DROPPED，代表事件不被丢弃
bool InputDispatcher::dispatchMotionLocked(
        nsecs_t currentTime, MotionEntry* entry, DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ATRACE_CALL();   //systrace抓取
    //*************1**************//
    // Preprocessing. 即标记当前已经进入分发的过程
    if (! entry->dispatchInProgress) {
        entry->dispatchInProgress = true;

        logOutboundMotionDetails("dispatchMotion - ", entry);    //log打印
    }

    //*************2**************//
   // Clean up if dropping the event. 如果事件是需要丢弃的，则返回true，不会去为该事件寻找合适的窗口
    if (*dropReason != DROP_REASON_NOT_DROPPED) {
        setInjectionResult(entry, *dropReason == DROP_REASON_POLICY
                ? INPUT_EVENT_INJECTION_SUCCEEDED : INPUT_EVENT_INJECTION_FAILED);
        return true;   //此时就是事件被丢弃了，分发任务就没有完成！
    }
    //**************3*************//
    bool isPointerEvent = entry->source & AINPUT_SOURCE_CLASS_POINTER;

    // 目标窗口信息列表会存储在inputTargets中
    std::vector<InputTarget> inputTargets;

    bool conflictingPointerActions = false;
    int32_t injectionResult;
    //事件处理的结果交由injectionResult
    if (isPointerEvent) {
        //1. 处理点击形式的事件，比如触摸屏幕
        injectionResult = findTouchedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
    } else {
        //2. 处理非触摸形式的事件，比如轨迹球
        injectionResult = findFocusedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime);
    }
    //**************4*************//
    //1. 如果injectionResult的值为INPUT_EVENT_INJECTION_PENDING，这说明找到了窗口并且窗口无响应输入事件被挂起，这时就会返回false
    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) {
        return false;
    }

    setInjectionResult(entry, injectionResult);
    //2. 如果injectionResult的值不为INPUT_EVENT_INJECTION_SUCCEEDED，这说明没有找到合适的窗口，输入事件没有分发成功，这时就会返回true
    //输入事件被挂起，说明找到了窗口并且窗口无响应
    if (injectionResult != INPUT_EVENT_INJECTION_SUCCEEDED) {
        if (injectionResult != INPUT_EVENT_INJECTION_PERMISSION_DENIED) {
            CancelationOptions::Mode mode(isPointerEvent ?
                    CancelationOptions::CANCEL_POINTER_EVENTS :
                    CancelationOptions::CANCEL_NON_POINTER_EVENTS);
            CancelationOptions options(mode, "input event injection failed");
            synthesizeCancelationEventsForMonitorsLocked(options);
        }
        return true;
    }

    //**************5*************//
    //分发目标添加到inputTargets列表中    // Add monitor channels from event's or focused display.
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(entry));

    if (isPointerEvent) {
        ssize_t stateIndex = mTouchStatesByDisplay.indexOfKey(entry->displayId);
        if (stateIndex >= 0) {
            const TouchState& state = mTouchStatesByDisplay.valueAt(stateIndex);
            if (!state.portalWindows.empty()) {
                // The event has gone through these portal windows, so we add monitoring targets of
                // the corresponding displays as well.
                for (size_t i = 0; i < state.portalWindows.size(); i++) {
                    const InputWindowInfo* windowInfo = state.portalWindows[i]->getInfo();
                    addGlobalMonitoringTargetsLocked(inputTargets, windowInfo->portalToDisplayId,
                            -windowInfo->frameLeft, -windowInfo->frameTop);
                }
            }
        }
    }

    // Dispatch the motion.
    if (conflictingPointerActions) {
        CancelationOptions options(CancelationOptions::CANCEL_POINTER_EVENTS,
                "conflicting pointer actions");
        synthesizeCancelationEventsForAllConnectionsLocked(options);
    }
    //将事件分发给inputTargets列表中的目标
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
......
```

### 4.4.1. InputTarget结构体

InputTarget结构体可以说是inputDispatcher与目标窗口的转换器。
其分为两大部分：

1. 一个是枚举中存储的inputDispatcher与目标窗口交互的标记，
2. 另一部分是inputDispatcher与目标窗口交互参数，比如：

+ `inputChannel`，它实际上是一个SocketPair，SocketPair用于进程间双向通信，这非常适合inputDispatcher与目标窗口之间的通信，因为inputDispatcher不仅要将事件分发到目标窗口，同时inputDispatcher也需要得到目标窗口对事件的响应。
+ `xOffset和yOffset`，屏幕坐标系相对于目标窗口坐标系的偏移量，MotionEntry(MotionEvent)中的存储的坐标是屏幕坐标系，因此就需要注释2和注释3处的参数，来将屏幕坐标系转换为目标窗口的坐标系。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.h
struct InputTarget {
  enum {
    //此标记表示事件正在交付给前台应用程序
    FLAG_FOREGROUND = 1 << 0,
    //此标记指示MotionEvent位于目标区域内
    FLAG_WINDOW_IS_OBSCURED = 1 << 1,
    ...
    };

    //inputDispatcher与目标窗口的通信管道
    sp<InputChannel> inputChannel;//1
    //事件派发的标记
    int32_t flags;
    //屏幕坐标系相对于目标窗口坐标系的偏移量
    float xOffset, yOffset;//2
    //屏幕坐标系相对于目标窗口坐标系的缩放系数
    float scaleFactor;//3
    BitSet32 pointerIds;
}
```

***

## 4.5. 处理点击事件findTouchedWindowTargetsLocked

> 参考：https://www.codercto.com/a/52484.html  
> 在函数dispatchMotionLocked中，会分别对Motion事件中的点击形式事件和非触摸形式事件做了处理。其中点击事件调用函数`findTouchedWindowTargetsLocked`。

函数末尾会打印类似日志`InputDispatcher: findTouchedWindow finished: injectionResult=0, injectionPermission=1, timeSpentWaitingForApplication=0.0ms`，injectionResult=0是succeed，injectionPermission=1是允许。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
int32_t InputDispatcher::findTouchedWindowTargetsLocked(nsecs_t currentTime,
        const MotionEntry* entry, std::vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime,
        bool* outConflictingPointerActions) {
    ATRACE_CALL(); //systrace
...
    if (newGesture || (isSplit && maskedAction == AMOTION_EVENT_ACTION_POINTER_DOWN)) {
        /* Case 1: New splittable pointer going down, or need target for hover or scroll. */
        //从MotionEntry中获取坐标点
        int32_t pointerIndex = getMotionEventActionPointerIndex(action);
        int32_t x = int32_t(entry->pointerCoords[pointerIndex].
                getAxisValue(AMOTION_EVENT_AXIS_X));
        int32_t y = int32_t(entry->pointerCoords[pointerIndex].
                getAxisValue(AMOTION_EVENT_AXIS_Y));
        bool isDown = maskedAction == AMOTION_EVENT_ACTION_DOWN;
        ...
        //将符合条件的窗口放入TempTouchState中，以便后续处理
        mTempTouchState.addOrUpdateWindow(newTouchedWindowHandle, targetFlags, pointerIds);
        }
        mTempTouchState.addGestureMonitors(newGestureMonitors);
    } else {
        /* Case 2: Pointer move, up, cancel or non-splittable pointer down. */

        ...

    //此处说明窗口已经查找成功
    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    //为每个mTempTouchState中的窗口生成InputTargets
    addWindowTargetLocked(focusedWindowHandle,
            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS, BitSet32(0),
            inputTargets);

    // Done.
        Unresponsive:
    // Reset temporary touch state to ensure we release unnecessary references to input channels.
    //重置TempTouchState
    mTempTouchState.reset();

    nsecs_t timeSpentWaitingForApplication = getTimeSpentWaitingForApplicationLocked(currentTime);
    updateDispatchStatistics(currentTime, entry, injectionResult, timeSpentWaitingForApplication);
#if DEBUG_FOCUS
    //日志打印输出
    ALOGD("findTouchedWindow finished: injectionResult=%d, injectionPermission=%d, "
            "timeSpentWaitingForApplication=%0.1fms",
            injectionResult, injectionPermission, timeSpentWaitingForApplication / 1000000.0);
#endif
    return injectionResult;
}
```

## 4.6. dispatchEventLocked向目标窗口发送事件

1. 上面函数dispatchMotionLocked的末尾，会执行`dispatchEventLocked`函数，将事件分发给inputTargets列表中的分发目标（目标窗口）。

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const std::vector<InputTarget>& inputTargets) {
    ATRACE_CALL();
#if DEBUG_DISPATCH_CYCLE
    ALOGD("dispatchEventToCurrentInputTargets");
#endif

    ALOG_ASSERT(eventEntry->dispatchInProgress); // should already have been set to true

    pokeUserActivityLocked(eventEntry);
    //遍历inputTargets列表，获取每一个inputTarget
    for (const InputTarget& inputTarget : inputTargets) {
        //1. 根据inputTarget内部的inputChannel来获取Connection的索引
        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            //2. 根据索引，获取保存在mConnectionsByFd容器中的Connection（可以理解为InputDispatcher和目标窗口的连接，其内部包含了连接的状态、InputChannel、InputWindowHandle和事件队列等）
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            //3. 根据inputTarget，开始事件发送循环
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        } else {
#if DEBUG_FOCUS
            ALOGD("Dropping event delivery to target with channel '%s' because it "
                    "is no longer registered with the input dispatcher.",
                    inputTarget.inputChannel->getName().c_str());
#endif
        }
    }
}
```

2. 开始事件发送，最终会通过inputTarget中的inputChannel和窗口进行`进程间通信`，最终将Motion事件发送给目标窗口。

```cpp
void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    if (ATRACE_ENABLED()) {
        std::string message = StringPrintf(
                "prepareDispatchCycleLocked(inputChannel=%s, sequenceNum=%" PRIu32 ")",
                connection->getInputChannelName().c_str(), eventEntry->sequenceNum);
        ATRACE_NAME(message.c_str());
    }
#if DEBUG_DISPATCH_CYCLE   //日志打印！
    ALOGD("channel '%s' ~ prepareDispatchCycle - flags=0x%08x, "
            "xOffset=%f, yOffset=%f, globalScaleFactor=%f, "
            "windowScaleFactor=(%f, %f), pointerIds=0x%x",
            connection->getInputChannelName().c_str(), inputTarget->flags,
            inputTarget->xOffset, inputTarget->yOffset,
            inputTarget->globalScaleFactor,
            inputTarget->windowXScale, inputTarget->windowYScale,
            inputTarget->pointerIds.value);
            ...
```

3. 然后调用startDispatchCycleLocked（在函数dispatchMotionLocked末尾处），最终调用两种事件的`connection->inputPublisher...`函数，至此，InputDisapatcher结束。

```cpp
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {
    if (ATRACE_ENABLED()) {
        std::string message = StringPrintf("startDispatchCycleLocked(inputChannel=%s)",
                connection->getInputChannelName().c_str());
        ATRACE_NAME(message.c_str());
    }
#if DEBUG_DISPATCH_CYCLE   //日志打印！
    ALOGD("channel '%s' ~ startDispatchCycle",
    ...
      case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);

            // Publish the key event.
            status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                    keyEntry->deviceId, keyEntry->source, keyEntry->displayId,
                    dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                    keyEntry->keyCode, keyEntry->scanCode,
                    keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                    keyEntry->eventTime);
            break;
        }
....
    case EventEntry::TYPE_MOTION: {
    // Publish the motion event.
            status = connection->inputPublisher.publishMotionEvent(dispatchEntry->seq,
                    motionEntry->deviceId, motionEntry->source, motionEntry->displayId,
                    dispatchEntry->resolvedAction, motionEntry->actionButton,
                    dispatchEntry->resolvedFlags, motionEntry->edgeFlags,
                    motionEntry->metaState, motionEntry->buttonState, motionEntry->classification,
                    xOffset, yOffset, motionEntry->xPrecision, motionEntry->yPrecision,
                    motionEntry->downTime, motionEntry->eventTime,
                    motionEntry->pointerCount, motionEntry->pointerProperties,
                    usingCoords);
            break;
        }
        ......
```
