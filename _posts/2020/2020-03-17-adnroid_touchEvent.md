---
layout: single
related: false
title:  Android 触控事件分析
date:   2020-03-17 22:32:00 +0800
categories: android
tags: input display
toc: true
---

> 我们常见的触摸事件除了按下，弹起，移动之外还有很多，诸如长按，双击，Scroll，Fling等，他们是怎么判断的，还有这些长按，双击等事件的时间能否自由设置。可以在开发者选项中打开“显示点按操作反馈”和“指针位置”，同时可以打开inputflinger模块的log开关做一些调试，分析TP报点。

一般当我们需要处理触摸事件时有两种方式：

+ 委托式 ： 将事件委托给监听器来进行处理。即定义一个`View.onTouchListener()`子类的监听器，由其`onTouch()`方法来处理。
+ 回调式 ： 通过重写View类自己的`onTouchEvent()`方法来处理，在执行时会回调该方法，在其中执行自定义的代码。

<!--more-->

**关于主触点，副触点**：发送触屏事件的时候，除了此触屏事件所对应的触点之外，如果当前触点多于一个或者等于一个，则此事件为副触点事件，发送此事件的触点叫做副触点。否则为主触点事件，发送此事件的触点为主触点。

# 1. MotionEvent对象事件处理

在`MotionEvent.java`中,ACTION动作事件定义

```java
ACTION_DOWN = 0;
ACTION_UP = 1;
ACTION_MOVE = 2;
ACTION_CANCEL =3 ;
ACTION_POINTER_DOWN = 5; //A non-primary pointer has gone down.
ACTION_POINTER_UP = 6;
ACTION_SCROLL = 8; //the most event contains relative vertical and/or horizontal scroll offset.
```

（1） 首先当点击下屏幕，触屏事件从`View.java`的onTouchEvent()开始处理：

```java
......
case MotionEvent.ACTION_DOWN:
    // a short period in case this is a scroll.
    if (isInScrollingContainer) {
        mPrivateFlags |= PFLAG_PREPRESSED;
        if (mPendingCheckForTap == null) {
            mPendingCheckForTap = new CheckForTap();
        }
        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
    } else {
        setPressed(true);
        checkForLongClick(0);
    }
break;
```

（2） 事件响应是先有按下才会有后续事件。因此先查看`ACTION_DOWN`。在此case中判断如果是在scrollingContainer中则等待一段时间执行检查是否为Tap事件。因为可能按下之后可能会有scroll操作，如果有将丢弃长按检测。而如果不在container中，则立即执行长按检测。

```java view.java
private final class CheckForTap implements Runnable {
        public void run() {
            mPrivateFlags &= ~PFLAG_PREPRESSED;
            setPressed(true);
            checkForLongClick(ViewConfiguration.getTapTimeout());
        }
 }
```

（3） 在其中执行了`setPressed()`操作，其后执行`checkForLongClick()`，即等待500ms-180ms 来执行longPress操作。

```java
postDelayed(mPendingCheckForLongPress,
         ViewConfiguration.getLongPressTimeout() - delayOffset);
```

在其中执行`performLongClick()`。在该函数中处理长按需要做的事情，例如长按监听器中流程，显示contextMenu，处理长按震动反馈：

```java
    handled = li.mOnLongClickListener.onLongClick(View.this);
    handled = showContextMenu();
    performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
```

**Note**: 此处有两个时间数据： `tapTimeout` 和 `longPressTimeout`。

定义在`frameworks/base/core/java/android/view/ViewConfiguration.java`，时间是可以自定义的，但最好采用google提供的，这是经过大量积累得来的数据。而此处的longTimeout是设置辅助功能界面中’触摸和按住延迟’选项可设置的，如果没有设置那就是用默认的500ms。

```java
private static final int TAP_TIMEOUT = 180;
private static final int DOUBLE_TAP_TIMEOUT = 300;
private static final int DEFAULT_LONG_PRESS_TIMEOUT = 500;
```

***

# 2. MotionEvent底层事件获取（触控事件分发机制）

（1） 在onResume时会将view显示出来，跟踪代码到执行时会调用`ActivityThread的handleResumeActivity()`。可以看到获取window的DecorView，即整个window的顶层View。
调用流程为:（创建窗口）

1. WindowManager.addView()；
2. 在实现类WindowManagerImpl中实现addView()；
3. 最后一行通过root.setView()；
4. 在ViewRootImpl中实现setView()；
5. 在其中调用windowSession.add()。
6. windowSession为客户端，而服务器端为`Session.java`,在Session中转而调用WindowManagerService的addWindow()来实现add方法。

（2）`WindowManagerService中addWindow`这里实现了事件信息传递和交互的通道，内部采用socketpair，通过`InputChannel`来实现。

**Note**：openInputChannelPair(), 在其中创建socketpair,一个匿名的已连接套接字，一个为发送端，一个为接收端，可以进行双工通讯（UNIX网络编程）。

获取InputChannel, 一个置为Input，一个置为output。RegisterInputChannel中调用nativeRegisterInputChannel。

（3）在`WindowManagerService`中创建InputManagerService类（InputManagerService.java）对象，并start。

之后通过JNI流程在native中执行，并执行InputManager的start方法。

（4）在创建InputReader时会将dispatcher传入。即InputReader的成员变量mQueuedListener为dispatcher的执行者，具体代码分析flush函数，关注Args，例如MotionArgs, flush执行后，将调用`dispatcher->notifyMotion()`;

如果只关注Motion的话，那么就是调用`InputDispatcher->notifyMotion()`。

**从抓取Systrace可以查看到触屏的整个事件，从InputReader开始，然后到deliverInputEvent触发APP绘制。关于报点可以重点关注inputflinger模块的log打印，会打印input的坐标。**

# 3. systrace查看Input事件流程

> 参考： https://www.jianshu.com/p/427b084b0d77
> 参考： https://mp.weixin.qq.com/s/Q2k6pLEyXhHZvZOIiU5ucA

1. 触摸屏每隔几毫秒（如果是60刷新率，则一秒扫描屏幕120次，大概8ms扫描一次）扫描一次，如果有触摸事件，那么把事件上报到对应的驱动。
2. InputReader 读取触摸事件交给 InputDispatcher 进行事件派发。
3. InputDispatcher 将触摸事件发给注册了 Input 事件的 App。
4. App 拿到事件之后，进行 Input 事件分发，如果此事件分发的过程中，App 的 UI 发生了变化，那么会请求 Vsync，则进行一帧的绘制。

## 3.1. 详细分析

所以systrace从InputReader开始：（前面还有一点很短的“binder transaction”的时间）

```java
//frameworks/native/services/inputflinger/InputReader.cpp
        // Dispatch pointer down events using the new pointer locations.
        while (!downIdBits.isEmpty()) {
            uint32_t downId = downIdBits.clearFirstMarkedBit();
            dispatchedIdBits.markBit(downId);

            if (dispatchedIdBits.count() == 1) {
                // First pointer is going down.  Set down time.
                mDownTime = when;
                /// M: for input MET systrace  @{
                ScopedTrace _l(ATRACE_TAG_INPUT, "AppLaunch_dispatchPtr:Down");
                /// @}
            }
```

然后会到InputDispatcher的dispatchMotionLocked函数，并且InputDispatcher会从InboundQueue中取出Input事件派发到各个App(连接)的OutBoundQueue(OutboundQueue区域oq)

```java
//frameworks/native/services/inputflinger/InputDispatcher.cpp
bool InputDispatcher::dispatchMotionLocked(
        nsecs_t currentTime, MotionEntry* entry, DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ATRACE_CALL();
    // Preprocessing.
    if (! entry->dispatchInProgress) {
        entry->dispatchInProgress = true;

        logOutboundMotionDetails("dispatchMotion - ", entry);
    }
```

然后到deliverInputEvent，说明APP UI Thread被Input事件唤醒；（起始点可以看到当前APP的Launcher是1，value=1表示有一个input事件，如果主线程卡顿没法及时处理Input事件，这里的Value会堆积）

之后则是APP的UI线程启动，然后再触发APP的绘制线程进行绘制等等。
