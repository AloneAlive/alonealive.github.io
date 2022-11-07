---
layout: single
related: false
title:  Android NE分析（二）
date:   2020-07-28 21:52:00
categories: android
tags: android debug
toc: true
---

> 了解gcc将`*.c/cpp`编译成`*.o`，再将其链接为可执行程序或/lib库的过程，有助于我们将native从编译/加载/执行到崩溃一条路贯通起来。Android的Makefile只需要将source file填入`LOCAL_SRC_FILES`，然后`include $(BUILD_SHARED_LIBRARY)`或`$(BUILD_EXECUTABLE)`就可以将`*.c/cpp/s`编译为动态库或可执行程序。

<!--more-->

# 1. native编译

## 1.1. 编译为obj

> 在build/core/definitions.mk有定义transform-c-or-s-to-o-no-deps和transform-cpp-to-o，分别将每个*.c/s和*.cpp编译成*.o，里面传了很多参数给gcc

1. ` -fpic -fPIE`

+ PIC是Position-Independent Code的缩写，经常被用在共享库中，这样就能将相同的库代码为每个程序映射到一个位置，不用担心覆盖掉其他程序或共享库。
+ PIE是Position-Independent-Executable的缩写，只能应用在可执行程序中。PIE和PIC很像，但做了一些调整（不用PLT，使用PC相关的重定位）。-fPIE给编译用，-pie给链接(ld)用。

例如，一个程序没有使用PIC被链接到0地址，那么系统将其加载到0地址。

2. `-fstack-protector`

> 顾名思义就是保护堆栈，每一个函数在运行时都有自己的栈帧，如果代码没有写好，很可能将自己甚至是其他的栈帧踩坏，那如何防护呢？简单的方法就是在栈帧头部也就是在局部变量开始之前多存储一个__stack_chk_guard值，用于在函数返回前取出来和_stack_chk_guard做对比，失败则调用__stack_chk_fail函数，这个就是该参数完成的行为。

## 1.2. 静态链接

`build/core/combo/TARGET_linux-arm.mk`里有定义`transform-o-to-static-executable-inner`，将*.o链接成静态可执行程序，静态可执行程序是一个完整的程序，不需要额外的共享库即可执行，比如/init,/sbin/adbd等。

链接器用的是arm-linux-androideabi-g++

## 1.3. 动态链接

`build/core/combo/TARGET_linux-arm.mk`里有定义`transform-o-to-executable-inner`和`transform-o-to-shared-lib-inner`，分别将*.o链接为动态可执行程序和共享库。动态可执行程序需要linker才能进一步运行的。

链接器也是用arm-linux-androideabi-g++

***

# 2. tombstone定位错误方法

## 2.1. signum

一般debuggerd关注的是SIGILL，SIGBUS，SIGABRT，SIGFPE，SIGSEGV，SIGPIPE等。而这里，估计九成都是SIGSEGV (即signal 11)，段错误，和非法内存访问等价。

## 2.2. *sigcode

+ SEGV_MAPERR：访问一个没有映射到任何内容的地址，这种情况通常就是野指针，或者越界访问，访问空指针也是属于这类
+ SEGV_ACCERR：试图访问您无权访问的地址。说明访问出错地址，被map到地址空间来了，但是没访问权限。基本上是指针越界或野指针，比如写只读map的内存地址

例如：

```log
signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0x7ecbf6a000   //SEGV_ACCERR表示试图访问您无权访问的地址
    x0  0000007e44001c30  x1  0000000000000000  x2  0000000000000000  x3  0000007e44001c30
    x4  0000007ecbf6a008  x5  0000007e44001c98  x6  0000000000000000  x7  0000000000000000
    x8  0000000000000000  x9  0000000000000000  x10 0000000000000000  x11 0000000000000000
    x12 0000000000000000  x13 0000000000000000  x14 0000000000000001  x15 0000007ecbe3f540
    x16 0000007eca917290  x17 0000007ec94ea480  x18 0000007e42e9c000  x19 0000007ecbdf5400
    x20 0000000000000001  x21 0000007e44001c30  x22 0000000000000001  x23 0000007e44002020
    x24 0000007ecb6bf045  x25 0000007ecb6bf260  x26 0000007ecb6bf278  x27 0000007ecb6bf041
    x28 0000007ecb6bf044  x29 0000007e44001bf0
    sp  0000007e44001bd0  lr  0000007eca910618  pc  0000007ec94ea460
```

tombstone日志当中也提供了出错时寄存器地址里面的临近内存信息，信息量同样很丰富。查看`0000007e44001c30`附近的内存情况。

```log
memory near x0:
    0000007e44001c10 0000000000030d40 00000000ffffffff  @...............
    0000007e44001c20 0000007e44001cf0 0000007ecb6b83f0  ...D~.....k.~...
    0000007e44001c30 0000000000000000 0000000000000000  ................
    0000007e44001c40 0000000000000000 0000000000000000  ................
    0000007e44001c50 0000000000000000 0000000000000000  ................
    0000007e44001c60 0000000000000000 0000000000000000  ................
    0000007e44001c70 0000000000000000 0000000000000000  ................
    0000007e44001c80 0000000000000000 0000000000000000  ................
    0000007e44001c90 0000000000000000 1c86a694ed72c72c  ........,.r.....
    0000007e44001ca0 0000000000000001 00000000000fd000  ................
    0000007e44001cb0 0000007ecb6b8170 0000007e44001d50  p.k.~...P..D~...
    0000007e44001cc0 0000007e44001d50 0000007e44001dd8  P..D~......D~...
    0000007e44001cd0 00000207000005ea 0000007e44001d50  ........P..D~...
    0000007e44001ce0 0000007ec954e5f0 0000007e44001d50  ..T.~...P..D~...
    0000007e44001cf0 0000007e44001d10 0000007ec954e618  ...D~.....T.~...
    0000007e44001d00 0000007e44001d50 0000000000000000  P..D~...........
```

# 3. 参考

+ Android Native/Tombstone Crash Log 详细分析：https://blog.csdn.net/u011006622/article/details/51496693
+ Android Native程序crash的一些定位方法简介：https://msd.misuland.com/pd/300217191876268032
+ ARM64-memcpy.S 汇编源码分析：https://blog.csdn.net/ffmxnjm/article/details/68065090
+ android bionic memcpy 汇编源码解析：https://blog.csdn.net/qq_28637193/article/details/103681746
+ PIC和PIE：https://www.cnblogs.com/sword03/p/9385660.html