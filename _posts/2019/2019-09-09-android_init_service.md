---
layout: single
related: false
title:  Android init进程
date:   2019-09-09 21:52:00
categories: android
tags: android
toc: true
---

> init是Linux系统中用户空间的第一个进程。Android底层也是和linux原理一样。

# 1. 概述

+ init是Linux系统中用户空间的第一个进程。通过`adb shell ps -rf`查看我的一加手机进程信息。

```shell
UID            PID  PPID C STIME TTY          TIME CMD
root             1     0 0 12:43:34 ?     00:00:10 init
```

+ init进程负责创建系统中的几个关键进程，例如zygote
+ init提供了一个property service（属性服务）来管理Android系统的众多属性

# 2. init分析

> 使用android-10.0.0_r2 AOSP最新源码（http://192.99.106.107:8080/xref/android-10.0.0_r2/）

init进程的入口函数从/system/core/init/main.cpp的main函数开始：

```cpp
// system/core/init/main.cpp
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#endif

    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap function_map;

            return SubcontextMain(argc, argv, &function_map);  
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv); //此处调用init.cpp的函数入口
        }
    }

    re
```

在Android Q之前，此处一直都是直接调用入口函数：

```cpp
// system/core/init/main.cpp
int main(int argc, char** argv) {
    android::init::main(argc, argv);
}
```

然后会调用到init.cpp中的SecondStageMain函数。init的工作流程精简为以下四点：
+ 解析配置文件
+ 执行各个阶段的动作（创建zygote的工作就是此时完成）
+ 调用Property_init初始化属性相关的资源
+ init进入一个无线循环，等待一些事情的发生（init处理来自socket和来自属性服务器的相关事情）

## 2.1. 解析系统配置文件init.rc

+ 在入口函数调用`LoadBootScripts(am, sm);`
+ 然后调用`parser.ParseConfig(bootscript);`
+ 调用system/core/init/parser.cpp的parseConfig函数
+ 调用`Parser::ParseConfigFile`函数，读取配置文件并解析

```cpp
bool Parser::ParseConfigFile(const std::string& path) {
    LOG(INFO) << "Parsing file " << path << "...";
    android::base::Timer t;
    auto config_contents = ReadFile(path);   //读取init.rc配置文件
    if (!config_contents) {
        LOG(INFO) << "Unable to read config file '" << path << "': " << config_contents.error();
        return false;
    }

    ParseData(path, &config_contents.value());

    LOG(VERBOSE) << "(Parsing " << path << " took " << t << ".)";
    return true;
}
```

## 2.2. init.rc解析service

> 查看/system/core/rootdir/init.rc

```cpp
on init   //on关键字表示一个section，对应的名字时init
    sysclktz 0

    # Mix device-specific information into the entropy pool
    copy /proc/cmdline /dev/urandom
    copy /system/etc/prop.default /dev/urandom

    symlink /proc/self/fd/0 /dev/stdin
...

# It is recommended to put unnecessary data/ initialization from post-fs-data
# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start && property:ro.crypto.state=unencrypted
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=unsupported
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on boot    //新的section，名为boot
    # basic network init
    ifup lo   //一个command
    hostname localhost
    domainname localdomain

    # IPsec SA default expiration length
    write /proc/sys/net/core/xfrm_acq_expires 3600
...
```

init.rc中：`class_start`，标识一个COMMAND，对应的处理函数是do_class_start，位于boot section范围内。

## 2.3. 属性服务

> 注册标可以存储一些类似ket/value的键值对，一般系统或某些应用程序会吧自己的一些属性存储在注册表中，即使重启，还是能够根据之前在注册表中设置的属性，进行相应的初始化工作。Android平台也提供了一个类似的机制，称为属性服务（property service）  

property的初始化在init.cpp的主函数里面：

```cpp
property_init();
......
    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);
...
    property_load_boot_defaults(load_debug_prop);
    UmountDebugRamdisk();
    fs_mgr_vendor_overlay_mount_all();
    export_oem_lock_status();
    StartPropertyService(&epoll);
    MountHandler mount_handler(&epoll);
    set_usb_controller();
......
```
