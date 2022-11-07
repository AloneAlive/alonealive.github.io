---
layout: single
related: false
title:  Android Input事件ANR流程
date:   2020-05-10 23:32:00
categories: android
tags: input display
toc: true
---

> Android Input体系中，大致有两种类型的事件：实体按键key事件，屏幕点击触摸事件。如果根据事件类型的不同，还能细分为基础实体按键的key(power，volume up/down，recents，back，home)，实体键按键，屏幕点击(多点，单点)，屏幕滑动等事件。在Android整个Input体系中有三个格外重要的成员：Eventhub，InputReader，InputDispatcher。它们分别担负着各自不同的职责，Eventhub负责监听`/dev/input`产生的Input事件；InputReader负责从Eventhub读取事件，并将读取的事件发给InputDispatcher；InputDispatcher则根据实际的需要具体分发给当前手机获得焦点实际的Window。  
> 常说的Input ANR超时，都是指的是Input事件分发超时。

<!--more-->

# 1. 问题日志

从下面的log可以看到超过了5s导致发生Input ANR事件。

+ main log：

```log
04-22 10:49:36.222646  1270  1376 I InputDispatcher: Application is not responding: AppWindowToken{e4f7c16 token=Token{8a1cd31 ActivityRecord{21295d8 u0 com.android.PACKAGE/.PACKAGE_Activity t10}}}.  It has been 5008.2ms since event, 5005.5ms since wait started.  Reason: Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.
04-22 10:49:36.342164  1270  1407 E TAG     : 82 Tanet
......
04-22 10:49:41.347174  1270  1376 I InputDispatcher: Dropped event because it is stale.
```

+ system log：

```log
04-22 10:49:31.216699  1270  1376 D PowerManagerService: getScreenOffTimeoutLocked:  isTestFlag = false
04-22 10:49:31.216986  1270  1376 W WindowManager: Failed looking up window callers=com.android.server.wm.InputManagerCallback.interceptKeyBeforeDispatching:182 com.android.server.input.InputManagerService.interceptKeyBeforeDispatching:1839 <bottom of call stack> 
04-22 10:49:36.224611  1270  1376 I WindowManager: Input event dispatching timed out .  Reason: Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.
04-22 10:49:36.341180  1270  1407 W InputManager: Input event injection from pid 6143 timed out.
04-22 10:49:36.342467  1270  1376 W WindowManager: Failed looking up window callers=com.android.server.wm.InputManagerCallback.interceptKeyBeforeDispatching:182
```

# 2. 代码分析

## 2.1. 循环读取分发Input事件

在frameworks/native/services/inputflinger/InputDispatcher.cpp中，流程从`InputDispatcherThread::threadLoop()`线程循环开始，方法体只调用循环一个函数`mDispatcher->dispatchOnce()`。

如果没有等待的命令，则会循环运行主要函数`dispatchOnceInnerLocked`不断的读取并分发Input事件：

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now(); //记录事件的当前时间点
...
    case EventEntry::TYPE_KEY: {   //点击事件
        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        if (isAppSwitchDue) {
            if (isAppSwitchKeyEvent(typedEntry)) {
                resetPendingAppSwitchLocked(true);
                isAppSwitchDue = false;
            } else if (dropReason == DROP_REASON_NOT_DROPPED) {
                dropReason = DROP_REASON_APP_SWITCH;
            }
        }
        if (dropReason == DROP_REASON_NOT_DROPPED
                && isStaleEvent(currentTime, typedEntry)) {
            dropReason = DROP_REASON_STALE;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED && mNextUnblockedEvent) {
            dropReason = DROP_REASON_BLOCKED;
        }
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);  //执行该函数
        break;
    }
...
    if (done) {
        if (dropReason != DROP_REASON_NOT_DROPPED) {
            dropInboundEventLocked(mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;

        releasePendingEventLocked();
        *nextWakeupTime = LONG_LONG_MIN;  // force next poll to wake up immediately
    }
}
```

该函数最后调用了dropInputEvent事件`dropInboundEventLocked(mPendingEvent, dropReason);`

## 2.2. 若case EventEntry::TYPE_KEY

如果是Key事件，则会执行`InputDispatcher::dispatchKeyLocked`函数。然后在该函数中调用`findFocusedWindowTargetsLocked`

```cpp
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
            ......
    // Identify targets.
    std::vector<InputTarget> inputTargets;
    int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
            entry, inputTargets, nextWakeupTime);
    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) {
        return false;
    }

    setInjectionResult(entry, injectionResult);
    if (injectionResult != INPUT_EVENT_INJECTION_SUCCEEDED) {
        return true;
    }

    // Add monitor channels from event's or focused display.
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(entry));

    // Dispatch the key.
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

在函数`findFocusedWindowTargetsLocked`中开始就会进行判断，当`focusedWindowHandle == nullptr`但是`focusedApplicationHandle != nullptr`的时候调用`handleTargetsNotReadyLocked`报出ANR的错误日志。


```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, std::vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
            ......
    // If there is no currently focused window and no focused application
    // then drop the event.
    if (focusedWindowHandle == nullptr) {
        if (focusedApplicationHandle != nullptr) {
            //monkey test的时候经常遇到类似log的ANR。典型的无窗口，有应用的ANR问题，这里我们就需要了解Android应用的启动流程了，一般此类问题都是Android应用首次启动时会发生此类问题，此时我们应用本身需要检查一下我们的Android应用重写的Application onCreate方法，Android应用的启动界面是否在onCreate onStart方法中是否存在耗时操作。当然不排除系统原因造成的启动慢，直接导致ANR问题发生的情况
            injectionResult = handleTargetsNotReadyLocked(currentTime, entry,
                    focusedApplicationHandle, nullptr, nextWakeupTime,
                    "Waiting because no window has focus but there is a "
                    "focused application that may eventually add a window "
                    "when it finishes starting up.");  
            goto Unresponsive;
        }
....
```

`InputDispatcher::handleTargetsNotReadyLocked`执行代码：

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
int32_t InputDispatcher::handleTargetsNotReadyLocked(nsecs_t currentTime,
        const EventEntry* entry,   //点击触摸事件
        const sp<InputApplicationHandle>& applicationHandle,
        const sp<InputWindowHandle>& windowHandle,
        nsecs_t* nextWakeupTime, const char* reason) {   
....
if (mInputTargetWaitCause != INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY) {
//这里一般是有应用（application已经创建），无窗口，或者有应用，有窗口ANR的情形，一般同一个窗口至进入一次该方法

            nsecs_t timeout;  //int64类型秒数
            if (windowHandle != nullptr) {
                timeout = windowHandle->getDispatchingTimeout(DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            } else if (applicationHandle != nullptr) {   //执行这个，有应用无窗口
                timeout = applicationHandle->getDispatchingTimeout(
                        DEFAULT_INPUT_DISPATCHING_TIMEOUT); //5s超时
            } else {
                timeout = DEFAULT_INPUT_DISPATCHING_TIMEOUT; //5s
            }

mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY;//超时等待原因
mInputTargetWaitStartTime = currentTime;//函数入参当前时间，此处就是当前input事件的第一次分发时间
mInputTargetWaitTimeoutTime = currentTime + timeout; //设置超时

            //无窗口
            if (windowHandle != NULL) {
                mInputTargetWaitApplicationHandle = windowHandle->inputApplicationHandle;//记录当前等待的应用
            }
            //TODO 记录当前等待的应用，针对无窗口，有应用
            if (mInputTargetWaitApplicationHandle == NULL && applicationHandle != NULL) {
                mInputTargetWaitApplicationHandle = applicationHandle;

...
    //当前时间已经大于超时时间，说明应用有时间分发超时了，需要触发ANR
    if (currentTime >= mInputTargetWaitTimeoutTime) {   //应该是超时5s
        onANRLocked(currentTime, applicationHandle, windowHandle,
                entry->eventTime , mInputTargetWaitStartTime, reason);

        // Force poll loop to wake up immediately on next iteration once we get the
        // ANR response back from the policy.
        *nextWakeupTime = LONG_LONG_MIN;
        return INPUT_EVENT_INJECTION_PENDING;
    } 
```

## 2.3. ANR的函数调用onANRLocked

最后发生ANR调用`InputDispatcher::onANRLocked`

```cpp
//frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::onANRLocked(
        nsecs_t currentTime, const sp<InputApplicationHandle>& applicationHandle,
        const sp<InputWindowHandle>& windowHandle,
        nsecs_t eventTime, nsecs_t waitStartTime, const char* reason) {
    float dispatchLatency = (currentTime - eventTime) * 0.000001f;
    float waitDuration = (currentTime - waitStartTime) * 0.000001f;
    ALOGI("Application is not responding: %s.  "
            "It has been %0.1fms since event, %0.1fms since wait started.  Reason: %s",
            getApplicationWindowLabel(applicationHandle, windowHandle).c_str(),
            dispatchLatency, waitDuration, reason);
```

***

# 3. 分析方法

1. 抓取systrace分析：可以分析Input事件的部分
2. 系统的Trace log：系统生成的Trace文件保存在`data/anr`,可以用过命令`adb pull data/anr/`取出。
3. 抓取日志分析

# 4. 可能导致ANR的原因

1. 怀疑是不是在Activty oncreate和onstart耗时太多，导致窗口还未创建好，input事件超时5s
应用窗口是在onResume中才去向WindowManager添加注册的。因此在注册添加窗口之前，application或者启动的Activity的生命周期onCreate，onStart的任意方法，做了耗时操作，或者他们加载一起的执行时间过长，都是能够导致`无窗口，有应用类型的Input ANR问题`发生的。所以实际开发应用的时候，就要尽可能的把耗时的操作，异步处理。具体异步实现思路可以使用`new thread + handler，Asynctask，HandlerThread`等等，这里推荐使用HandlerThread，因为google封装的接口，使用起来简单。
2. 可能是UI主线程做了耗时的操作。

# 5. 参考

+ 参考：https://blog.csdn.net/abm1993/article/details/80461752
+ 参考：https://blog.csdn.net/abm1993/article/details/80497039
+ 参考：https://www.jianshu.com/p/f05d6b05ba17
