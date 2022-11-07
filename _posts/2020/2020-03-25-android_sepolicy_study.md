---
layout: single
related: false
title:  Android SELinux权限笔记
date:   2020-03-20 23:32:00 +0800
categories: selinux
tags: selinux android
toc: true
---

> 在新增一个HIDL Service后，需要对其进行权限配置，不然通过`adb shell ps -A|grep NAService`会发现该service无法启动，也会通过抓取log发现一堆的`avc: denied`权限问题。关于SELinux可以推荐文档：https://www.pianshen.com/article/6549296922/， 非常详细，本文只是作为记录部分常用的笔记。

<!--more-->

> `Android sepolicy`，SEAndroid是一种基于安全策略的MAC安全机制。SEAndroid安全机制中的安全策略就是在安全上下文的基础上进行描述的，也就是说，它通过主体和客体的安全上下文，定义主体是否有权限访问客体。

> 例如添加一个service，在`.rc`文件定义了service，还需要在`sepolicy`的`file_context`中添加权限。

# 1. selinux相关命令

```s
// 查看进程的sContext
ps -Z

// 查看文件权限
ls -Z
```

查看selinux开关状态：`adb shell getenforce`

可能返回结果有三种：Enforcing、Permissive 和 Disabled。Disabled 代表 SELinux 被禁用，Permissive 代表仅记录安全警告但不阻止可疑行为，Enforcing 代表记录警告且阻止可疑行为。

一般调试通过以下命令关闭SELinux权限（需重启生效）：

```s
adb root
adb shell setenforce 0
```

## 1.1. 抓取SELinux Log

1. 抓kernel log，`adb shell dmesg`
2. 抓kernel log，使用命令,可以直接提出avc的log：`adb shell "cat /proc/kmsg | grep avc" > avc_log.txt `
3. `adb logcat –b events`,搜索关键字：`avc: denied`

***

# 2. File_contexts

> 用于声明文件的安全上下文，plat前缀的文件用于声明system、rootfs、data等与设备无关的文件。Nonplat 用于声明vendor、data/vendor等文件。

## 2.1. domain.te

> 该策略文件会限制一些特征文件的权限，一般不建议修改。

## 2.2. selinux没有对某个文件的权限（有neverAllow）处理方法

> 参考：https://blog.csdn.net/ly890700/article/details/54645212

```log
01-01 08:03:22.410000   217   217 W applypatch: type=1400 audit(0.0:16): avc: denied { read } for name="mmcblk0p15" dev="tmpfs" ino=3364 scontext=u:r:install_recovery:s0 tcontext=u:object_r:block_device:s0 tclass=blk_file permissive=0  
```

意思是说明`install_revovery`没有block_device的权限

只要在install_recovery.te中加入下面权限就可以了。
+ `allow install_recovery recover_block_device:blk_file { open read write }; `

***

# 3. Service_contexts

> 用于声明java service 的安全上下文， O上将该文件拆分为`plat`和`nonplat`前缀的两个文件，但nonplat前缀的文件并没有具体的内容（vendor和system java service不允许binder操作）。

# 4. Property_contexts

> 用于声明属性的安全上下文，plat 前缀的文件用于声明system属性，nonplat前缀的文件用于声明vendor 属性。ril.开头的属性的安全上下文为`u:object_r:radio_prop:s0`，这意味着只有有权限访问Type为radio_prop的资源的进程才可以访问这些属性。

# 5. Hwservice_contexts

新增文件，用于声明HIDL service 安全上下文。
```s
android.hardware.vibrator::IVibrator                u:object_r:hal_vibrator_hwservice:s0
android.hardware.vr::IVr                            u:object_r:hal_vr_hwservice:s0
android.hardware.weaver::IWeaver                    u:object_r:hal_weaver_hwservice:s0
android.hardware.wifi::IWifi                        u:object_r:hal_wifi_hwservice:s0
android.hardware.wifi.hostapd::IHostapd             u:object_r:hal_wifi_hostapd_hwservice:s0
android.hardware.wifi.offload::IOffload             u:object_r:hal_wifi_offload_hwservice:s0
android.hidl.allocator::IAllocator                  u:object_r:hidl_allocator_hwservice:s0
android.hidl.base::IBase                            u:object_r:hidl_base_hwservice:s0
android.hidl.manager::IServiceManager               u:object_r:hidl_manager_hwservice:s0
android.hidl.memory::IMapper                        u:object_r:hidl_memory_hwservice:s0
android.hidl.token::ITokenManager                   u:object_r:hidl_token_hwservice:s0
android.system.net.netd::INetd                      u:object_r:system_net_netd_hwservice:s0
```

# 6. te语法

+ `allow signal`：

```s
allow domain domain : process signal; # 每个进程都能向它自己和其它进程发送signal  
allow domain self : process signal;   # 每个进程都能向它自己发送signal
```

***

> 参考文档：
+ https://blog.csdn.net/ch853199769/article/details/82501078
+ https://blog.csdn.net/innost/article/details/19299937
