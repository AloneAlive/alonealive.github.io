---
layout: single
related: false
title:  Android zygote和SystemServer进程
date:   2019-09-10 22:52:00
categories: android
tags: android
toc: true
---

> zygote和system_server在Android中的Java层很重要。

# 1. zygote分析

> zygote由init进程根据init.rc的配置项创建的。最初叫`app_process`，但是在运行过程中，通过Linux的pctrl系统调用将其换成了`zygote`。通过`adb shell ps -ef|grep zygote`查看到该进程。

Android-10.0.0_r2 AOSP源码中，查看其入口函数：

```cpp
// frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
    if (!LOG_NDEBUG) {
      String8 argv_String;
      for (int i = 0; i < argc; ++i) {
        argv_String.append("\"");
        argv_String.append(argv[i]);
        argv_String.append("\" ");
      }
      ALOGV("app_process main with argv: %s", argv_String.string());
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;
......
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

重要的功能由AppRuntime的start函数完成。而AppRuntime类就在app_main.cpp中，从AndroidRuntime派生而来。

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");
......
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {    //创建虚拟机
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {    //注册JNI函数
        ALOGE("Unable to register all android natives\n");
        return;
    }
.....
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);   //通过JNI调用Java中ZygoteInit的main函数，进入Java世界

......
```

## 1.1. 创建虚拟机startVM

> 该函数调用JNI的Dalvik虚拟机创建函数，在sdtartVM中确定创建虚拟机的一些参数

## 1.2. 注册JNI函数 startReg

> 给虚拟机注册一些JNI函数，采用native方式实现。

## 1.3. java入口 CallStaticVoidMethod

> CallStaticVoidMethod最终调用`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`的main函数。

```cpp
    @UnsupportedAppUsage
    public static void main(String argv[]) {
        ZygoteServer zygoteServer = null;

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        Runnable caller;
......
  if (startSystemServer) {    //启动system_server
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }
......
```

`forkSystemServer`会创建java世界中系统service所驻留的进程system_server，该进程是framework的核心。

***

# 2. SystemServer

SystemServer的进程名叫做`system_server`，由zygote进程中创建。

`forkSystemServer`函数中调用`handleSystemServerProcess()来处理自己的事务`。

调用到systemserver的main函数。

```cpp
// frameworks/base/services/java/com/android/server/SystemServer.java
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }

    public SystemServer() {
        // Check for factory test mode.
        mFactoryTestMode = FactoryTest.getMode();

        // Record process start information.
        // Note SYSPROP_START_COUNT will increment by *2* on a FDE device when it fully boots;
        // one for the password screen, second for the actual boot.
        mStartCount = SystemProperties.getInt(SYSPROP_START_COUNT, 0) + 1;
        mRuntimeStartElapsedTime = SystemClock.elapsedRealtime();
        mRuntimeStartUptime = SystemClock.uptimeMillis();

        // Remember if it's runtime restart(when sys.boot_completed is already set) or reboot
        // We don't use "mStartCount > 1" here because it'll be wrong on a FDE device.
        // TODO: mRuntimeRestart will *not* be set to true if the proccess crashes before
        // sys.boot_completed is set. Fix it.
        mRuntimeRestart = "1".equals(SystemProperties.get("sys.boot_completed"));
    }
```

随后会创建一些系统服务，并将调用线程加入到Binder通信中。并且会创建一个单独的线程，用以启动系统的各项服务，例如电池管理服务BatteryService，电源管理服务PowerManagerService，StartWindowManagerService，ActivityManagerService等等。

# 3. 开机耗时长的原因

1. ZygoteInit的main函数中的preloadClasses加载了上千个类
2. 开机启动时会对系统所有的APK扫描并收集信息
3. SystemServer创建的一系列service，占用不少时间

# 4. 虚拟机heapsize的限制

zygote创建虚拟机的时候，系统默认设置的java虚拟机堆栈值（可修改）对于使用较大内存的程序远远不够。zygote通过fork创建子进程，因而本身设置的信息会被子进程全部继承，例如设置堆栈对32MB，则子进程也会使用32MB。

# 5. watchdog看门狗

watchdog作用是每隔一段时间去检查某个参数是否被设置了，如果发现该参数没有被设置，则判断为系统出错，然后强制重启。

Android对于systemserver的参数是否被设置也增加了一条看门狗。主要检查几个重要的service，如果service出了问题就会杀掉system_server，这回导致zygote也一起杀掉，导致java层重启。

SystemServer和Watchdog的交互大致分为三个步骤(frameworks/base/services/core/java/com/android/server/Watchdog.java)：
1. Watchdog.getInstance().init()初始化
2. Watchdog.getInstance().start()，派生于Thread类，start启动线程，导致Watchdog的run在另外一个线程中被执行。该函数实现隔一段时间发送一条信息，那个线程将检查各个service的健康状况，而看门狗等待检查结果，如果第二次没有返回结果，将会杀掉systemserver
3. Watchdog.getInstance().addMonitor()，如果要支持看门狗的检查，就需要让service实现monitor接口（比如ActivityManagerService,PowerManagerService,WindowManagerService）

**Example:**

当一个函数占着锁，长时间没有返回（原因是这个函数需要和硬件交互，而硬件没有及时返回），导致系统服务死锁被watchdog检查到。

