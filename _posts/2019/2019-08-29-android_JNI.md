---
layout: single
related: false
title:  Android JNI理解
date:   2019-08-29 23:41:06
categories: android
tags: JNI
toc: true
---

> JNI,即Java Native Interface，Java本地调用。

# 1. 概述

通过JNI可以实现：  
> + Java程序函数可以调用Natvie语言（C/C++）写的函数
> + Natvie程序函数可以调用Java层的函数
<!--more-->

# 2. MediaScanner示例

> 使用Android Xref提供的 [Android 9.0.0_r3的源码](http://androidxref.com/9.0.0_r3/)

## 2.1. Java层的MediaScanner

完成两件事：
1. 加载JNI库
2. Java的native函数

```cpp
// frameworks/base/media/java/android/media/MediaScanner.java
public class MediaScanner implements AutoCloseable {
    static {
        System.loadLibrary("media_jni");  //加载对应的JNI库，media_jni是库名，在实际加载动态库的时候会拓展成libmedia_jni.so
        native_init();  //调用函数
    }

    private final static String TAG = "MediaScanner";
...
    private native void processDirectory(String path, MediaScannerClient client);
    private native boolean processFile(String path, String mimeType, MediaScannerClient client);
    private native void setLocale(String locale);

    public native byte[] extractAlbumArt(FileDescriptor fd);
    private static native final void native_init();
    private native final void native_setup();
    private native final void native_finalize();

```

动态库是运行时加载的库。如果Java要调用native函数，必须通过一个位于JNI层的动态库实现。通常是在类的static语句中加载，调用`System.loadLibrary`方法就可以加载。

函数名前有Java的关键字`native`的函数表示将由JNI层实现。

因而Java层只需要两项工作：**加载对应的JNI库，和声明由关键字native修饰的函数**

## 2.2. JNI层的MediaScanner

JNI对应的文件是`frameworks/base/media/jni/android_media_MediaScanner.cpp`

```cpp
// frameworks/base/media/jni/android_media_MediaScanner.cpp
// This function gets a field ID, which in turn causes class initialization.
// It is called from a static block in MediaScanner, which won't run until the
// first time an instance of this class is used.
static void
android_media_MediaScanner_native_init(JNIEnv *env)
{
    ALOGV("native_init");
    jclass clazz = env->FindClass(kClassMediaScanner);
    if (clazz == NULL) {
        return;
    }

    fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
    if (fields.context == NULL) {
        return;
    }
}
...
```

java层的native_init函数对应`android_media_MediaScanner_native_init`。通过文件路径来命名，观察两个文件的路径：
+ frameworks/base/media/java/android/media/MediaScanner.java
+ frameworks/base/media/jni/android_media_MediaScanner.cpp

### 2.2.1. 注册JNI函数

java层的MediaScanner.java函数native_init位于android.media包中，全路径名是：`android/media/MediaScanner.java`，对应JNI层函数的名字。JNI层将Java函数名称（包含包名）中的`.`转换成`_`，通过这种方式，native_init对应JNI的函数。

JNI函数注册的意思是将java层的native函数和JNI层对应的实现函数关联起来。注册有两种方式：`静态方法和动态注册`

1.静态注册

> 根据函数名来找对应的JNI函数。

（1） 编写Java代码，编译生成.class文件

（2） 使用Java的工具命令`javah -o output packagename.classname`，生成一个`output.h`的JNI头文件，里面声明了对应的JNI函数，只要实现里面的函数即可。

2.动态注册

> 因为Java native函数和JNI函数是一一对应的，所以存在一种`JNINativeMethod`的结构记录这种一一对应的关系。

例如frameworks/base/media/jni/android_media_MediaScanner.cpp文件中：

```cpp
...
static const JNINativeMethod gMethods[] = {
 ......
    {
        "native_setup",
        "()V",
        (void *)android_media_MediaScanner_native_setup
    },

    {
        "native_finalize",
        "()V",
        (void *)android_media_MediaScanner_native_finalize
    },
};
//注册上面的数组
// This function only registers the native methods, and is called from
// JNI_OnLoad in android_media_MediaPlayer.cpp
int register_android_media_MediaScanner(JNIEnv *env)
{
    return AndroidRuntime::registerNativeMethods(env,
                kClassMediaScanner, gMethods, NELEM(gMethods));
}
```

当Java层通过`System.loadLibrary`加载完JNI动态库后，会查找该库中`JNI_OnLoad`函数，然后调用他，之后完成动态注册。

```cpp
jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
{
    JNIEnv* env = NULL;
    jint result = -1;

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        ALOGE("ERROR: GetEnv failed\n");
        goto bail;
    }
    assert(env != NULL);

......
    if (register_android_media_MediaRecorder(env) < 0) {
        ALOGE("ERROR: MediaRecorder native registration failed\n");
        goto bail;
    }

    if (register_android_media_MediaScanner(env) < 0) {   //此处开始动态注册
        ALOGE("ERROR: MediaScanner native registration failed\n");
        goto bail;
    }

    if (register_android_media_MediaMetadataRetriever(env) < 0) {
        ALOGE("ERROR: MediaMetadataRetriever native registration failed\n");
        goto bail;
    }
......

    /* success -- return valid version number */
    result = JNI_VERSION_1_4;

bail:
    return result;
}
```
