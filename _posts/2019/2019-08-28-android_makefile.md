---
layout: single
related: false
title:  Android中的makefile（Android.mk）
date:   2019-08-28 21:52:00
categories: android
tags: makefile
toc: true
---

> makefile是和make命令一起使用，在Android中，可以使用`mm`、`mmm`、`mma`进行编译。  
> Makefile可以组织项目中各种库和代码之间的依赖，构建项目，和maven、gradle一样属于构建工具。  
> 常用于大型项目。
<!--more-->

# 1. 基本语法

## 1.1. 变量定义`=或者:=`

```shell
OBJS = programA.o
//或者
OBJS := programA.o
```

两者区别在于`:=`只能使用前面定义好的变量，`=`可以使用后面定义的变量。

## 1.2. 变量值追加`+=`

```shell
SRCS := programB.c
SRCS += programC.c
```

# 2. makefile在Android中的运用

name|note
:-:|:-:
LOCAL_PATH = $(call my-dir) | 调用my-dir函数，返回Android.mk文件所在的目录，放在第一行，地址是当前目录
include file Makefile | 引入其他的makefile文件
include $(CLEAR_VARS) | 编译模块时清空LOCAL_MODULE等参数
LOCAL_MODULE | 模块名称
LOCAL_SRC_FILES | 编译需要的源文件
LOCAL_C_INCLUDES | 需要的头文件
LOCAL_SHARED_LIBRARIES | 编译需要的动态库
LOCAL_LDLIBS | 链接库

***

## 2.1. 引入aidl文件

```shell
LOCAL_SRC_FILES := $(call all-laidl-files-under, src/com/srm/aidl)

或者直接如下，但是最后要加上 \ 符号，并且符号之后要回车：
LOCAL_SRC_FILES += $(call all-java-files-under, src) \
src/com/srm/aidl/test.aidl \

```

## 2.2. 添加jar包（libs和mk同目录）

```shell
LOCAL_STATIC_JAVA_LIBRARIES := testjar

//需要用CLEAR_VARS分割
include $(CLEAR_VARS)
LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := testjar:libs/testjar.jar

//需要添加
include $(BUILD_MULTI_PREBUILT)
```
