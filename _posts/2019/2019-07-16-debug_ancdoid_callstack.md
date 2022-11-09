---
layout: single
related: false
title: 在Android源码调试函数的堆栈
date:   2019-07-16 20:04:14
categories: android
tags: debug
toc: true
---

> 在Android代码中可以通过了解函数的CallStack加速调试和分析过程，本文说明如何在Android运行时加入CallStack及Android P上的注意点。

# 1. Java

```java
import android.util.Log; 
Log.d("yourTag", Log.getStackTraceString(new Exception()));
```

# 2. C++

> Android 9以前CallStack call是被build进libutils, framework大部分service都是link了该lib，因此可以直接使用Callstack。  
> Android 9开始后CallStack被build进libutilscallstack，因此直接使用Callstack会报`undefined reference to ‘android::CallStack::CallStack 在Android.bp或Android.mk中加入”libutilscallstack”` 即可.

```cpp
#include <utils/CallStack.h>

ALOGD("dump callstack");
android::CallStack stack; 
stack.update(); 
stack.dump("yourTag");
// stack.log("yourTag"); //callstack LOG_TAG

Methods:
adb logcat | grep yourTag
```
