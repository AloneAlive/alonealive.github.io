---
layout: single
related: false
title:  Android Property属性概念
date:   2019-10-16 23:52:00
categories: android
tags: android
toc: true
---

> 讲述Android property属性分类、服务、生成结果文件

# 1. property的相关生成文件和设置

> android通过SystemProperties的set和get方法来控制很多东西，一般上层添加一个控制开关可以使用这个方法，在系统里面存在很多个prop文件。它们分别是`system/build.prop`,`system/etc/prop.default`,`vendor/build.prop`,`vendor/default.prop`。下面分别来说下这几个文件的构成。

+ system/build.prop

这个主要是由device\***(platform)sample\product/system.prop,还有在build目录下添加的ADDITIONAL_BUILD_PROPERTIES

+ system/etc/prop.default

主要是系统添加的PRODUCT_SYSTEM_DEFAULT_PROPERTIES

+ vendor/build.prop（比较重要）

主要是系统添加的PRODUCT_PROPERTY_OVERRIDES，添加在device.mk的这个属性会被编译到这里，但是在9.0的系统，加到这里会无效，获取不到值。

+ vendor/default.prop（会被同目录的build.prop相同property覆盖）

主要是系统添加的PRODUCT_DEFAULT_PROPERTY_OVERRIDES

***

# 2. Android property属性

+ 参考：[Android 添加系统属性](https://blog.csdn.net/u014674293/article/details/120670723)

在 Android 系统中有一个 Property Service 服务，这个服务对外提供了两个接口：

+ SystemProperties.get(String key, String def) 读取系统属性
+ SystemProperties.set(String key, String val) 设置系统属性

**特殊前缀属性：**
+ ro：只读属性，不能修改。
+ persist：修改属性后，重启依然有效。数据会保存到 /data/property 目录。其他前缀的属性被设置后，只是保存在内存中而已，并没有保存到磁盘，所有重启后就恢复默认值了。
+ ctrl：用来启动和停止服务。每一项服务必须在 init.rc 中定义。init 一旦收到设置 ctrl.start 属性的请求，属性服务将使用该属性值作为服务名找到该服务，启动该服务。这项服务的启动结果将会放入 init.svc.<服务名> 属性中。
