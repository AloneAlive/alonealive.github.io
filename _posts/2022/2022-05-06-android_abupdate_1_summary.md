---
layout: single
title:  Android AB升级（一） - 升级各层级模块概述
date:   2022-05-06 14:19:02 +0800 
categories: ota 
tags: android ota AB升级
toc: true
---

> Android A/B升级又称静默升级，它是一种在系统运行过程中进行的升级功能。为了减小系统运行负荷，整个升级过程会保持在一个较低的IO状态，所以升级时间比recovery升级明显要长。

# 1. 升级方法

当AB系统升级时，有两种方式来调用updateengine,来实现升级：
+ 一种方法是直接执行shell命令，调用update_engine_client，带参数来实现升级。界面不会显示，可通过打印日志观察进度`adb logcat -s update_engine`

```shell
//解压升级包
adb shell
update_engine_client --payload=file:///storage/5F49-FB9D/socupdate8g/payload.bin --update --headers="FILE_HASH=YP7Z1bFDv6O8C5LTWZ20JxTljXyoVitlCX27TBTyVDM=
FILE_SIZE=967460335
METADATA_HASH=1gpTz/Q7T1ysTu6suP8N2KVOfa+vKEdnJGnPsKcPiXw=
METADATA_SIZE=75378"
```

+ 另一种方式是应用层直接调用UpdateEngine的`applyPayload`方法来升级，主要的核心逻辑模块是update_engine

***

# 2. 原生Demo APP

> 原生的升级参考示例APK（以Android Q AOSP为例）：`packages/apps/Car/SystemUpdater`

这是Android P（9）Google提供的一个Demo APK，可以用作本地U盘测试升级

启动方式：`adb shell am start com.android.car.systemupdater/.SystemUpdaterActivity`

**该应用大概流程逻辑：**
1. 读取U盘中的升级文件，用户点击目标升级文件
2. 调用updateEngine传递主要参数，updateEngine的callback向用户显示升级升级进度
3. 在升级结束之后通知powermanager重启reboot机器

***

# 3. 应用层API相关接口说明

## 3.1. framwork层应用接口

**源代码位置：**
+ `framwork/base/core/java/android/os/UpdateEngine.java`
+ `framwork/base/core/java/android/os/UpdateEngineCallback.java`

## 3.2. APP应用调取升级接口applyUpdate流程

（需要系统权限的App，需要系统签名，这些Api也是@SystemApi的）

1. 创建UpdateEngineCallback的对象mUpdateEngineCallback
2. 创建UpdateEngine的对象mUpdateEngine, 创建后服务开启
3. 使用`mUpdateEngine.bind(mUpdateEngineCallback)`，因为bind方法时接受的callback对象，而我们创建的类继承了callback,传入当前类的对象即可
4. 调用`applyPayload(String url,long offset,long size,String[] headerKeyValuePairs)`方法具体执行升级
5. 在重写的`onStatusUpdate(int status, float percent)`方法中根据拿到的状态执行进度逻辑
6. 在重写的`onPayloadApplicationComplete(int errorCode)`方法中执行升级完成后的逻辑

***

# 4. update_engine模块概述

> update_engine是A/B升级的核心逻辑，用于执行下载和升级，路径：`android/system/update_engine/`
> 
> update_engine_client：update_engine_client是客户端进程，用来解析命令行的各种操作(`suspend/resume/cancel/reset_status/follow/update`)，并将这些操作和参数通过binder机制，转发为对服务端进程UpdateEngineService相应操作的调用

## 4.1. update engine核心Action流程

update_engine升级会依次执行四个升级动作：`InstallAction，DownloadAction，FilesystemVerfierAction，PostinstallRunnerAction`

启动DownloadAction和PostinstallRunnerAction耗时最长

1. DownloadAction是执行具体的升级拷贝动作，将镜像文件中的内容拷贝到指定分区，这一步的时间不容易缩减。

2. PostinstallRunnerAction是根据post install script进行预编译工作，将target_slot标记为active。

在device.mk配置是：

```s
#调试工具
PRODUCT_PACKAGES_DEBUG += \
    update_engine_client

# A/B OTA dexopt package升级脚本
PRODUCT_PACKAGES += otapreopt_script

#将这部分注释掉，可以减少PostinstallRunnerAction步骤的耗时（需谨慎修改）
# A/B OTA dexopt update_engine hookup
AB_OTA_POSTINSTALL_CONFIG += \
    RUN_POSTINSTALL_system=true \
    POSTINSTALL_PATH_system=system/bin/otapreopt_script \
    FILESYSTEM_TYPE_system=ext4 \
    POSTINSTALL_OPTIONAL_system=true
```

***

# 5. 参考

+ [OTA升级系列](https://www.cnblogs.com/startkey/category/1416503.html)