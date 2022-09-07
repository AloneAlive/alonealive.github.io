---
layout: single
title:  Android 动态分区相关模块和常用工具
date:   2022-08-09 13:58:02 +0800
categories: ota
tags: android ota AB升级
toc: true
---

> Android动态分区功能编译和调试用到的lpmake、lpdump、lpunpack、dmctl等命令工具，以及涉及到的一些库模块，比如liblp、lipdm等。

# 1. 动态分区映射

## 1.1. super.img编译和生成

在Android中系统通过脚本`build/tools/releasetools/build_super_image.py`内部去调用`lpmake`工具生成super.img镜像

所以，在编译的log中查找lpmake就直接看到系统是如何去生成super.img的

```shell
//build/tools/releasetools/build_super_image.py
def BuildSuperImageFromDict(info_dict, output):

  cmd = [info_dict["lpmake"],
         "--metadata-size", "65536",
         "--super-name", info_dict["super_metadata_device"]]

  ab_update = info_dict.get("ab_update") == "true"
  retrofit = info_dict.get("dynamic_partition_retrofit") == "true"
  block_devices = shlex.split(info_dict.get("super_block_devices", "").strip())
  groups = shlex.split(info_dict.get("super_partition_groups", "").strip())

  if ab_update and retrofit:
    cmd += ["--metadata-slots", "2"]
  elif ab_update:
    cmd += ["--metadata-slots", "3"]
  else:
    cmd += ["--metadata-slots", "2"]
    ......
```

***

### 1.1.1. lpmake命令编译super.img

参考脚本，可以直接用命令生成super.img，不用管各种动态分区配置

```shell
lpmake --metadata-size 65536 --super-name super --metadata-slots 3 \
    --device super:3028287488 \
	--group bcm_ref_a:1509949440 --group bcm_ref_b:1509949440 \
	--partition system_a:readonly:1077702656:bcm_ref_a \
	--image system_a=out/target/product/productName/system.img \
	--partition system_b:readonly:0:bcm_ref_b \
	--partition vendor_a:readonly:104992768:bcm_ref_a \
	--image vendor_a=out/target/product/productName/vendor.img \
	--partition vendor_b:readonly:0:bcm_ref_b \
	--sparse --output out/target/product/productName/super.img
```

***

## 1.2. super.img解析

由于上面命令生成super.img中带有`--sparse`选项，所以生成的`super.img`为`sparse image`格式，在解析前需要使用工具`simg2img`将其转换成raw格式。

`simg2img out/target/product/productName/super.img out/super_raw.img`

对super.img进行简单的解析，`lpdump`工具已经足够了：

```shell
$ lpdump out/super_raw.img 
Metadata version: 10.0
Metadata size: 592 bytes
Metadata max size: 65536 bytes
Metadata slot count: 3
Partition table:
------------------------
  Name: system_a
  Group: bcm_ref_a
  Attributes: readonly
  Extents:
    0 .. 2104887 linear super 2048
------------------------
  Name: system_b
  Group: bcm_ref_b
  Attributes: readonly
  Extents:
------------------------
  Name: vendor_a
  Group: bcm_ref_a
  Attributes: readonly
  Extents:
    0 .. 205063 linear super 2107392
------------------------
  Name: vendor_b
  Group: bcm_ref_b
  Attributes: readonly
  Extents:
------------------------
Block device table:
------------------------
  Partition name: super
  First sector: 2048
  Size: 3028287488 bytes
  Flags: none
------------------------
Group table:
------------------------
  Name: default
  Maximum size: 0 bytes
  Flags: none
------------------------
  Name: bcm_ref_a
  Maximum size: 1509949440 bytes
  Flags: none
------------------------
  Name: bcm_ref_b
  Maximum size: 1509949440 bytes
  Flags: none
------------------------
```

**解析的结果:**

+ 1个device: super
+ 3个group: “default”, “bcm_ref_a” 和 “bcm_ref_b”
+ 4个partition: “system_a”, “system_b”, “vendor_a”, “vendor_b”, 其中 “system_b” 和 “vendor_b” 在super.img中没有镜像文件，因为没有将这两个分区的“image”参数传递给lpmake命令
+ 2个extent: 分别对应于“system_a”和“vendor_a”的镜像文件，lpdump的结果直接将extents结果附加到对应分区了，所以没有单独列出extents

**Android代码中，这部分操作通过调用库`liblp`内的函数完成，结束后在内存中生成`LpMetadata`结构数据**

***

## 1.3. super.img映射

1. 在解析super.img生成LpMetadata结构以后

2. `libdm`库内的函数会基于分区partitions和条带extents内的信息创建映射表

对于system_a和vendor_a分区，分别有以下映射:

```shell
/* system_a */
{
    0, 2104888,
    "/dev/block/by-id/super",
    2048
},

/* vendor_a */
{
    0, 205064,
    "/dev/block/by-id/super",
    2107392
}
```

**换个表述方式就是：**

+ 将 “/dev/block/by-id/super” 分区的 2048 开始的 2104888 个 sector 映射到 “/dev/block/mapper/system_a” 设备的 0 位置。
+ 将 “/dev/block/by-id/super” 分区的 2107392 开始的 205064 个 sector 映射到 /dev/block/mapper/vendor_a 设备的 0 位置

3. 然后linux系统的device mapper驱动基于每个设备的映射表生成相应的虚拟设备，这样就可以和虚拟出来的 “system_a”, “vendor_a” 分区进行使用，而不用管这些设备到底是真实的还是虚拟的

***

## 1.4. 小结——动态分区生成、编译、映射流程

1. 编译阶段build_super_image.py内部调用lpmake工具生成super.img文件
2. Android启动时系统通过liblp库函数解析super.img头部的metadata，在内存中建立LpMetadata数据结构
3. fs_mgr基于LpMetadata内的分区信息，得到映射表，然后通过libdm库调用linux的device mapper驱动映射设备

***

# 2. 动态分区相关模块

## 2.1. liblp

liblp模块代码位于`system/core/fs_mgr/liblp”`目录中

其主要功能是负责动态分区数据在内存和物理设备上的增删改查操作。

动态分区核心数据结构的定义位于以下两个文件中:

```shell
system/core/fs_mgr/liblp/include/liblp/metadata_format.h
system/core/fs_mgr/liblp/include/liblp/liblp.h
```

***

## 2.2. libdm

libdm模块代码位于`system/core/fs_mgr/libdm`目录中

libdm就是device mapper library的意思，libdm对`linux device mapper ioctl`操作接口进行封装，将其封装成Device Mapper对象，以对象的方式提供给其它模块使用。

libdm基于分区映射表，调用`Linux device mapper`驱动接口，实现对虚拟设备的创建，删除和状态查询工作

例如上面对镜像的两个映射表

libdm 的作用就是基于这两个映射表，将其提交给 linux 的 device mapper 驱动生成虚拟设备 system_a 和 vendor_a，除了可以创建虚拟设备之外，也可以删除虚拟设备或查询虚拟设备的状态

***

## 2.3. libfs_mgr

libfs_mgr模块代码位于`system/core/fs_mgr`目录中

liblp模块负责读取设备上的metadata数据到内存建立LpMetadata结构，而libdm模块负责对虚拟设备的创建、删除和查询操作。

**而这中间就是libfs_mgr模块，它的功能之一就是基于liblp获取的LpMetadata创建映射表，并将映射表传递给libdm模块，用来创建虚拟设备**

libfs_mgr是整个Android分区和文件系统管理的模块，动态分区管理只是其中一个功能，相关的代码也只有一个文件:

`system/core/fs_mgr/fs_mgr_dm_linear.cpp`

***

## 2.4. libsparse

libsparse模块代码位于`system/core/libsparse`目录中

libsparse主要用于对sparse文件的处理，只有当需要处理sparse image格式的image时才会需要用到这个库的函数

**libsparse模块下有三个工具:**

### 2.4.1. simg2img和img2simg

这两个工具以源码形式提供，编译Android时会将这两个工具生成到`out/host/linux-x86/bin`目录下。其作用是将image镜像文件在sparse image和raw image间转换

### 2.4.2. simg_dump.py

python脚本工具，专门用来查看sparse image的信息，如果想分析sparse image的结构，可以参考这个脚本

***

# 3. 动态分区相关工具

**动态分区分析调试中可能用到的工具包括：**

1. 位于`system/extras/partition_tools`目录下的动态分区工具lpmake, lpdump, lpflash, lpunpack 和lpadd，其中lpadd在Android R及其以后版本才有
2. 位于`system/core/fs_mgr/tools`目录下的device mapper工具dmctl和dmuserd，其中dmuserd在Android R及其以后版本才有
3. 位于`system/core/libsparse`目录下的sparse image文件处理工具 simg2img, img2simg, append2simg, simg_dump.py

## 3.1. lpmake

> 可以查看文档`system/extras/partition_tools/README.md`中的介绍，了解使用方法

Android 编译时，build_super_image.py脚本会准备命令并调用lpmake生成super.img，直接在Android编译的log中搜索lpmake就可以看到详细的命令

可以参考上面生成super.img的命令自行调整去生成需要的动态分区文件

***

## 3.2. lpdump

lpdump可以用来分析super分区头部的metadata信息，对metadata进行简单分析时经常使用，参数简单，操作方便，可以分别生成在host和device上运行的版本

```shell
$ lpdump --help
lpdump - command-line tool for dumping Android Logical Partition images.

Usage:
  lpdump [-s <SLOT#>|--slot=<SLOT#>] [-j|--json] [FILE|DEVICE]

Options:
  -s, --slot=N     Slot number or suffix.
  -j, --json       Print in JSON format.
```

### 3.2.1. 项目编译代码中使用场景

```shell
$ simg2img out/target/product/inuvik/super.img out/super_raw.img
$ lpdump out/super_raw.img 
```

**在代码上分析super.img时，默认生成super.img是raw image格式，因此在分析前需要先将其转换成raw image格式**

### 3.2.2. 设备使用场景

```shell
lpdump -s 0 /dev/block/by-name/super
```

***

## 3.3. lpunpack

lpmake命令负责将多个分区文件打包成动态分区文件super.img，lpunpack则相反，可以将super.img拆分，得到每个分区各自的image

```shell
$ simg2img super.img out/super_raw.img
$ lpunpack super_raw.img unpack/
$ ls -lh unpack/
total 1.1G
-rw-r--r-- 1 rg935739 users 1.1G Apr  2 15:09 system_a.img
-rw-r--r-- 1 rg935739 users    0 Apr  2 15:09 system_b.img
-rw-r--r-- 1 rg935739 users 101M Apr  2 15:09 vendor_a.img
-rw-r--r-- 1 rg935739 users    0 Apr  2 15:09 vendor_b.img
```

**注意：在使用lpunpack解包前，也需要先将super.img从sparse image转换成raw image**

***

## 3.4. dmctl

dmctl是Android上用来操作调试分区映射的很好用的工具，功能很多，具体使用`dmctl help`查看使用方法

## 3.5. dmsetup

跟dmctl是Android上专用的工具相比，dmsetup是x86机器上一个通用的管理device mapper虚拟设备的工具，可以在host上执行`dmsetup --help`看下使用方法

以下是操作示例，操作的重点有两个:

1. 先使用losetup将super_raw.img映射到loop设备
2. 将loop设备内的指定区域映射到虚拟设备“/dev/mapper/dm-rocky”

```shell
# 1. 将 super.img 转换成 raw 格式
$ simg2img target/product/inuvik4/super.img super_raw.img

# 2. 将 super_raw.img 文件挂载为一个 loop 设备
$ sudo losetup -f super_raw.img

# 3. 查看 super_raw.img 映射的 loop 设备，这里映射成了 /dev/loop2
$ losetup -l
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                            DIO LOG-SEC
/dev/loop1         0      0         1  1 /var/lib/snapd/snaps/lxd_22526.snap   0     512
/dev/loop2         0      0         0  0 /public/android-r/out/super_raw.img   0     512

# 4. 构建映射表
# 前面 3.4 节使用 dmctl 操作时的映射表如下:
# "<target_type> <start_sector> <num_sectors> <block_device>           <physical_sector>"
# " linear        0              2498103       /dev/block/by-name/super 2048"
# 由于 dmsetup 的参数有些变化，所以将上面的参数修改为 dmsetup 的参数格式:
# "<logical_start_sector num_sectors target_type destination_device start_sector>"
# " 0                    2104359     linear      /dev/loop2         2048"
# 因为这里的 /dev/loop2 相当于 Android 上的 /dev/block/by-name/super, 所以适当修改

# 5. 使用 dmsetup 将 "super_a" 映射成虚拟设备 "dm-rocky"
$ sudo dmsetup create dm-rocky --table '0 2104359 linear /dev/loop2 2048'

# 6. 查看映射的虚拟设备 dm-rocky 及其相关信息
$ ls -lh /dev/mapper/
total 0
crw------- 1 root root 10, 236 Mar 12 01:01 control
lrwxrwxrwx 1 root root       7 Apr  2 16:50 dm-rocky -> ../dm-5
$ sudo dmsetup table dm-rocky
0 2104359 linear 7:2 2048
$ sudo dmsetup info dm-rocky
Name:              dm-rocky
State:             ACTIVE
Read Ahead:        256
Tables present:    LIVE
Open count:        0
Event number:      0
Major, minor:      253, 5
Number of targets: 1

# 7. 检查下 dm-rocky 虚拟设备的内容和 super_raw.img 文件内的"system_a"数据是否一样,
# 如果二者的 md5 值一直，则内容一样
# 这里检查 super_raw.img 内容是使用 dd 命令获取相应部分，然后传递给 md5sum   
$ sudo md5sum /dev/mapper/dm-rocky 
71a8376a44ba321d9033f3ccae83f277  /dev/mapper/dm-rocky
$ dd if=super_raw.img skip=2048 bs=512 count=2104359 | md5sum
2104359+0 records in
2104359+0 records out
1077431808 bytes (1.1 GB, 1.0 GiB) copied, 1.63181 s, 660 MB/s
71a8376a44ba321d9033f3ccae83f277  -

# 二者的 md5 值一样，说明 /dev/mapper/dm-rocky 映射的就是 super 分区的 "system_a" 部分

# 8. 使用 dmseutp 删除映射的虚拟设备 "dm-rocky"
$ sudo dmsetup remove dm-rocky

# 9. 取消 super_raw.img 对 /dev/loop2 的 loop 设备映射
$ sudo losetup -d /dev/loop2
$ losetup -l /dev/loop2
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE DIO LOG-SEC
/dev/loop2                          0  0             0     512
```

***

## 3.6. dmuserd

从Android R开始引入dmuserd工具，这个工具和虚拟A/B 分区(Virtual A/B) 有关

***

# 4. 小结

+ Android 动态分区最核心的处理模块主要有3个: liblp, libdm, libfs_mgr, libsparse
+ Android 动态分区文件处理包括: lpmake, lpdump, lpunpack, lpflash, lpadd，工具使用说明参考文档: system/extras/partition_tools/README.md
+ Android 动态分区映射工具:dmctl,dmuserd
+ Linux上通用的虚拟分区映射工具:dmsetup

***

# 5. 参考

+ [Android 动态分区详解(一) 5 张图让你搞懂动态分区原理](https://blog.csdn.net/guyongqiangx/article/details/123899602)
+ [Android 动态分区详解(二) 核心模块和相关工具介绍](https://blog.csdn.net/guyongqiangx/article/details/123931356)