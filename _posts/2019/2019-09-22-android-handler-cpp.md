---
layout: single
related: false
title:  Android Handler消息循环处理机制(例ActivityThread)
date:   2019-09-22 15:01:32
categories: android
tags: android
toc: true
---

> Android Handler消息循环处理机制(例ActivityThread)

# 1. 整体流程

> 关键词：Handler、Looper、MessageQueue、handleMessage

消息被存放于消息队列，应用程序的主线程会围绕这个消息队列进入一个无限循环，知道应用程序退出。（消息循环过程是由Looper实现的）
+ 如果队列中有消息，应用程序的主线程会把它取出来，分发给相应的Handler进行处理；
+ 如果队列中没有消息，应用程序的主县城就会进入空闲等待状态，等待下一个消息的到来；

***

# 2. 消息循环（以ActivityThread为例）

应用程序的消息循环是从 ActivityThread 的 main()函数入口的，在 main()函数中会调用`Looper.prepareMainLooper();`和`Looper.loop();`

> 代码： http://aosp.opersys.com/xref/android-10.0.0_r2/xref/frameworks/base/core/java/android/app/ActivityThread.java#7310

```java
//frameworks/base/core/java/android/app/ActivityThread.java
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // Install selective syscall interception
        AndroidOs.install();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
        // It will be in the format "seq=114"
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();  //消息循环

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

创建Looper对象的时候，会同时创建一个 MessageQueue，保存在 Looper 的成员变量 mQueue 中。Looper和MessageQueue就是这样关联起来的。

JNI层创建 Looper 时会通过 pipe 系统调用来创建一个管道。

```java
//frameworks/base/core/java/android/os/Looper.java
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

该Looper对象创建好之后会保存在 NativeMessageQueue 对象的成员变量 mLooper 中，这个对象的作用是，通过管道实现以下功能：当Java层的消息队列中没有消息时，就使 Android 应用程序主线程进入等待状态，而当Java层的消息队列中来了消息时，就唤醒Android应用程序的主线程来处理这个消息。

ActivityThread调用`Looper.loop()`才会进入消息循环。

进入消息循环后，会不断地从消息队列`mQueue`中去获取下一个要处理的消息msg，如果消息的target为null，就表示要退出消息循环了，否则就会调用target对象的 dispatchMessage 函数来处理这个消息。

调用 queue 的 next 函数去获取下一个要处理的消息，但调用这个函数有可能会让线程进入等待状态。一是当消息队列中没有消息时，它会使线程进入等待状态；二是消息队列中有消息，但是消息指定了执行的时间，而现在还没有到这个时间，线程也会进入等待状态。

```java
//frameworks/base/core/java/android/os/Looper.java
   public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            // Make sure the observer won't change while processing a transaction.
            final Observer observer = sObserver;

            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            Object token = null;
            if (observer != null) {
                token = observer.messageDispatchStarting();
            }
            long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
            try {
                msg.target.dispatchMessage(msg);
                if (observer != null) {
                    observer.messageDispatched(token, msg);
                }
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } catch (Exception exception) {
                if (observer != null) {
                    observer.dispatchingThrewException(token, msg, exception);
                }
                throw exception;
            } finally {
                ThreadLocalWorkSource.restore(origWorkSource);
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```

调用 queue 的 next 函数去获取下一个要处理的消息，但调用这个函数有可能会让线程进入等待状态。一是当消息队列中没有消息时，它会使线程进入等待状态；二是消息队列中有消息，但是消息指定了执行的时间，而现在还没有到这个时间，线程也会进入等待状态。

![消息处理流程](../../assets/post/2019/2019-09-22-android-handler-cpp/Handler-1.png)

***

# 3. 消息发送

从应用程序启动入口分析下消息发送流程。应用程序启动过程中会调用 sendMessage 函数向应用程序的消息队列中加入一个新的消息。
sendMessage将参数封装成Message，然后通过mH.sendMessage把该消息加入消息队列。mH是ActivityThread类的成员变量，它的类型为H，继承于`Handler`类。

定义： `final H mH = new H();`

```java
//frameworks/base/core/java/android/app/ActivityThread.java
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) {
            Slog.v(TAG,
                    "SCHEDULE " + what + " " + mH.codeToString(what) + ": " + arg1 + " / " + obj);
        }
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```

sendMessage函数是继承于Handler的，Handler在它的构造函数中获取了Looper对象和MessageQueue 对象。

```java
//frameworks/base/core/java/android/os/Handler.java
    public final boolean sendMessage(@NonNull Message msg) {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

```

`msg.target = this;`表示这个消息最终由这个Handler对象来处理，即由ActivityThread对象的mH成员变量来处理。最终会调到MessageQueue的`enqueueMessage`函数最后会调到`Looper.cpp`的`wake`函数。

```java
//frameworks/base/core/java/android/os/Handler.java
    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

```java
//frameworks/base/core/java/android/os/MessageQueue.java
 boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);   //此处调用JNI函数
            }
        }
        return true;
    }
```

```cpp
//frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
```

***

# 4. 消息处理

> ActivityThread的main函数在`Looper.loop`函数中调用`msg.target.dispatchMessage(msg)`去处理消息。

```java
//frameworks/base/core/java/android/os/Looper.java
  try {
                msg.target.dispatchMessage(msg);   //处理消息
                if (observer != null) {
                    observer.messageDispatched(token, msg);
                }
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } catch (Exception exception) {
                if (observer != null) {
                    observer.dispatchingThrewException(token, msg, exception);
                }
                throw exception;
            } finally {
                ThreadLocalWorkSource.restore(origWorkSource);
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
```

消息对象msg的成员变量`target`是在发送消息的时候设置好的，通过哪个Handler来发送消息，就通过哪个Handler来处理消息。

当时是 H 类把消息加入消息队列的，现在也该由 H 类处理消息。 H类没有实现自己的`dispatchMessage`函数，但它继承了父类Handler的dispatchMessage函数。

```java
//frameworks/base/core/java/android/os/Handler.java
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

```

最终调用`handleMessage`处理消息。此处既是调用`ActivityThread.java`的该函数。

```java
//frameworks/base/core/java/android/app/ActivityThread.java
  public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                    .......
```

***

# 5. SurfaceFlinger的消息处理机制

> 类似的消息处理机制在SurfaceFlinger也存在，拥有独自的文件`frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp`等待消息、发送消息、处理消息。从而进行Layer合成事件。

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
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
    }
}
```
