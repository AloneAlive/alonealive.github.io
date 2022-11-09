---
layout: single
related: false
title:  Android @hide接口注释
date:   2019-11-03 12:32:00
categories: android
tags: android
toc: true
---

> Android @hide接口注释概念和使用方式

# 1. @hide和{@hide}

类或API是否开放是通过doc的注释｛＠hide｝来控制的

比如android.media.Metadata这个类就是android没有公开的类

因为在frameworks\base\media\libmedia\Metadata.java文件中，定义Metadata类之前有/**｛＠hide｝*/doc注释，所以Metadata类被定义为了非公开类，即在android应用程序中无法直接访问的类

**google 给了两个选择：**

在你添加的API或者变量前面增加javadoc 注释＠hide，但是要注意并不是简单写个＠hide或者 /*@hide*/就可以了，这些都是错误的javadoc注释格式。标准的javadoc都是这样的 /** */ 而且对于 format 变量应该加上 { }，所以我们应该这样写/** {@hide} */

想要生成的javadoc里面出现这个方法或者变量，你必须输入:`make update-api`。但是如果修改的是google没有开放出来的类，比如RIL、PhoneFactory，就不会出现这个问题。

***

# 2. 访问被@hide的API（android 如何引用@hide（隐藏）的类，方法和常量）

## 2.1. 直接将@hide标记去掉，将重新编译了的android.jar包换掉

不过强烈的建议不要这样做，别人隐藏起来的类或者方法肯定是不安全的，如果你把@hide放出来可能引起一些程序不可预知的错误。

## 2.2. 利用反射机制使用@hide方法，这种方法在网上看到一篇不错的，简单易懂，要深入的自己再到网上搜

## 2.3. 修改android.mk文件

删除LOCAL_SDK_VERSION := current

## 2.4. 将LOCAL_SDK_VERSION 注释掉之后提到服务器编译出现了代码混淆错误。

这个时候可以在android.mk文件中将LOCAL_PROGUARD_ENABLED := disabled加上。LOCAL_PROGUARD_ENABLED := disabled不使用代码混淆的工具进行代码混淆,如果不设置，默认使用LOCAL_PROGUARD_ENABLED := full.即将该工程代码全部混淆。

***

# 3. Android 10的变化

在Android 10对非SDK接口进行了限制，因而`@hide`注释的方法被列入黑名单，外部不能访问。

但是可以通过`adb shell settings put global hidden_api_policy  1`命令打开权限访问。

通过`adb shell settings delete global hidden_api_policy`解除设置。（https://developer.android.google.cn/about/versions/10/non-sdk-q?hl=en）