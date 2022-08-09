---
layout: single
title:  Android 动态分区配置原生示例
date:   2022-08-09 16:58:02 +0800
categories: android
tags: android
toc: true
---

> 动态分区分为原生动态分区和改造动态分区两种配置方式，其中包含开关配置和参数配置，以Android Q源码给出的原生示例为参考。

# 1. 动态分区配置

## 1.1. 原生动态分区配置

```shell
#文件device.mk
# 动态分区总开关
PRODUCT_USE_DYNAMIC_PARTITIONS := true

#文件BoardConfig.mk
# 设置 super 分区大小
BOARD_SUPER_PARTITION_SIZE := <size-in-bytes>

# 设置分区组, 可以设置多个组，对于 A/B 设备，每组最终会有 _a 和 _b 两个 slot
# 这里以分区组 group_foo 为例，会生成 group_foo_a 和 group_foo_b 两个组
BOARD_SUPER_PARTITION_GROUPS := group_foo

# 设置分区组包含的分区, 这里包含 system, vendor 和 product 等 3 个分区
BOARD_GROUP_FOO_PARTITION_LIST := system vendor product

# 设置分区组总大小, 总大小需要能够放下分区组里面的所有分区
BOARD_GROUP_FOO_SIZE := <size-in-bytes>

# 启用块级重复信息删除，可以进一步压缩 ext4 映像
BOARD_EXT4_SHARE_DUP_BLOCKS := true
```

***

## 1.2. 改造动态分区配置

对于改造动态分区(retrofit), 需要以下设置:

+ 注意：改造动态分区时，需要通过`BOARD_SUPER_PARTITION_METADATA_DEVICE`指定metadata存放的分区。因此，不仅可以将metadata数据和某个分区放到一起，例如原生动态分区中就是将metadata和super分区放到一起；也可以将 metadata数据单独放到某个分区中

```shell
#文件device.mk
# 改造(retrofit)动态分区总开关, 这里多了一个 retrofit，标明是升级改造设备
PRODUCT_USE_DYNAMIC_PARTITIONS := true
PRODUCT_RETROFIT_DYNAMIC_PARTITIONS := true

#文件BoardConfig.mk
# 设置为所有动态分区内子分区大小的总和
BOARD_SUPER_PARTITION_SIZE := <size-in-bytes>

# 设置动态分区子分区, 这里包含 system, vendor 和 product 等 3 个分区
BOARD_SUPER_PARTITION_BLOCK_DEVICES := system vendor product

# 逐个设置每一个子分区大小, 这设置 system, vendor 分区大小 
# BOARD_SUPER_PARTITION_$(partition)_DEVICE_SIZE
BOARD_SUPER_PARTITION_SYSTEM_DEVICE_SIZE := <size-in-bytes>
BOARD_SUPER_PARTITION_VENDOR_DEVICE_SIZE := <size-in-bytes>

# 设置分区组, 可以设置多个组，每组最终会有 _a 和 _b 两个 slot
BOARD_SUPER_PARTITION_GROUPS := group_foo

# 设置分区组包含的分区, 这里包含 system, vendor 和 product 等 3 个分区
BOARD_GROUP_FOO_PARTITION_LIST := system vendor product

# 设置分区组总大小, 总大小需要能够放下分区组里面的所有分区
BOARD_GROUP_FOO_SIZE := <size-in-bytes>

# 指定 metadata 数据存放的设备，这里设置为 system 分区，也可以是单独的分区
BOARD_SUPER_PARTITION_METADATA_DEVICE := system

# 启用块级重复信息删除，可以进一步压缩 ext4 映像
BOARD_EXT4_SHARE_DUP_BLOCKS := true
```

***

## 1.3. 注意事项

1. 在动态分区配置中，不再需要以下分区大小设置了，例如：

```shell
BOARD_SYSTEMIMAGE_PARTITION_SIZE := 4294967296 # 4 GB
BOARD_VENDORIMAGE_PARTITION_SIZE := 536870912 # 512MB
BOARD_PRODUCTIMAGE_PARTITION_SIZE := 1610612736 # 1.5GB
```

2. 原生动态分区中，super分区内的system, vendor, product等需要从GPT分区表中移除
3. 应避免将userdata, cache或任何其他永久性读写分区放在super分区中

***

# 2. 动态分区配置示例

> 关于动态分区配置，这里再以三个AOSP自带的google设备动态分区配置为例说明，包括原生动态分区和改造动态分区(retrofit)，这部分配置位于`device/google`目录之下

## 2.1. crosshatch 设备(Pixel 3 XL)配置示例

**crosshatch 设备(Pixel 3 XL) 支持原生动态分区，也支持改造动态分区，配置如下：**

```shell
#文件device/google/crosshatch/BoardConfig-common.mk
#启用动态分区宏开关
ifneq ($(PRODUCT_USE_DYNAMIC_PARTITIONS), true)
  # ...
else
  BOARD_EXT4_SHARE_DUP_BLOCKS := true
endif

ifeq ($(PRODUCT_USE_DYNAMIC_PARTITIONS), true)
BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions
BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := \
    system \
    vendor \
    product

#改造动态分区配置
ifeq ($(PRODUCT_RETROFIT_DYNAMIC_PARTITIONS), true)
# Normal Pixel 3 must retrofit dynamic partitions.
BOARD_SUPER_PARTITION_SIZE := 4072669184
BOARD_SUPER_PARTITION_METADATA_DEVICE := system
BOARD_SUPER_PARTITION_BLOCK_DEVICES := system vendor product
BOARD_SUPER_PARTITION_SYSTEM_DEVICE_SIZE := 2952790016
BOARD_SUPER_PARTITION_VENDOR_DEVICE_SIZE := 805306368
BOARD_SUPER_PARTITION_PRODUCT_DEVICE_SIZE := 314572800
# Assume 4MB metadata size.
# TODO(b/117997386): Use correct metadata size.
BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 4069523456

#原生动态分区配置
else
# Mainline Pixel 3 has an actual super partition.

# TODO (b/136154856) product_services partition is removed.
# Instead, we will add system_ext once it is ready.
# BOARD_PRODUCT_SERVICESIMAGE_FILE_SYSTEM_TYPE := ext4
# TARGET_COPY_OUT_PRODUCT_SERVICES := product_services

BOARD_SUPER_PARTITION_SIZE := 12884901888
# Assume 1MB metadata size.
# TODO(b/117997386): Use correct metadata size.
BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 6441402368

# TODO (b/136154856) product_services partition removed.
# Instead, we will add system_ext once it is ready.
# BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST += \
#    product_services \

endif # PRODUCT_RETROFIT_DYNAMIC_PARTITIONS
endif # PRODUCT_USE_DYNAMIC_PARTITIONS
```

crosshatch 动态分区总体上，设备定义了 1 个动态分区组`google_dynamic_partitions`, 包含分区 system vendor product。

**对于原生动态分区，有：**

```shell
# 启用块级重复信息删除，可以进一步压缩 ext4 映像
BOARD_EXT4_SHARE_DUP_BLOCKS := true

# 总开关
PRODUCT_USE_DYNAMIC_PARTITIONS := true
# 分区组和子分区
BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions
BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product
# super 分区和分区组大小
BOARD_SUPER_PARTITION_SIZE := 12884901888
BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 6441402368
```

**对于改造动态分区，有:**

```shell
# 启用块级重复信息删除，可以进一步压缩 ext4 映像
BOARD_EXT4_SHARE_DUP_BLOCKS := true

# 总开关
PRODUCT_USE_DYNAMIC_PARTITIONS := true
PRODUCT_RETROFIT_DYNAMIC_PARTITIONS := true
# 分区组和子分区
BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions
BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product
# super 分区大小
BOARD_SUPER_PARTITION_SIZE := 4072669184
# metadata 存放的设备
BOARD_SUPER_PARTITION_METADATA_DEVICE := system
# 动态分区内的子分区
BOARD_SUPER_PARTITION_BLOCK_DEVICES := system vendor product
# 每个子分区大小
BOARD_SUPER_PARTITION_SYSTEM_DEVICE_SIZE := 2952790016  # 2816M
BOARD_SUPER_PARTITION_VENDOR_DEVICE_SIZE := 805306368   # 768M
BOARD_SUPER_PARTITION_PRODUCT_DEVICE_SIZE := 314572800  # 300M
# 分区组大小
BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 4069523456
```

***

## 2.2. bonito设备(Pixel 3a XL)配置示例（改造动态分区）

bonito设备(Pixel 3a XL)只支持改造动态分区，配置如下：

**从这里的配置看，和 crosshatch 设备(Pixel 3 XL)对改造动态分区的配置是一样的，只是少了一个product分区**

```shell
# 文件device/google/bonito/device-common.mk
# Enable retrofit dynamic partitions for all bonito
# and sargo targets
PRODUCT_USE_DYNAMIC_PARTITIONS := true   //启用动态分区
PRODUCT_RETROFIT_DYNAMIC_PARTITIONS := true  //改造动态分区

# 文件device/google/bonito/BoardConfig-common.mk
BOARD_EXT4_SHARE_DUP_BLOCKS := true  //启用块级重复信息删除
BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions  //分组
BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := \
    system \
    vendor \
    product

BOARD_SUPER_PARTITION_SIZE := 4072669184
BOARD_SUPER_PARTITION_METADATA_DEVICE := system
BOARD_SUPER_PARTITION_BLOCK_DEVICES := system vendor
BOARD_SUPER_PARTITION_SYSTEM_DEVICE_SIZE := 3267362816
BOARD_SUPER_PARTITION_VENDOR_DEVICE_SIZE := 805306368
# Assume 4MB metadata size.
# TODO(b/117997386): Use correct metadata size.
BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 4068474880
```

***

## 2.3. 模拟器cuttlefish配置示例（原生动态分区）

模拟器cuttlefish的动态分区配置位于文件:`device/google/cuttlefish/shared/BoardConfig.mk`，如下：

```shell
# device/google/cuttlefish/shared/BoardConfig.mk

# 使用自定义的TARGET_USE_DYNAMIC_PARTITIONS 作为开关，而不是PRODUCT_USE_DYNAMIC_PARTITIONS, 不过后者会根据前者设置为true
# ============= 参考device.mk配置
ifeq ($(TARGET_USE_DYNAMIC_PARTITIONS),true)
  PRODUCT_USE_DYNAMIC_PARTITIONS := true
  TARGET_BUILD_SYSTEM_ROOT_IMAGE := false
else
  TARGET_BUILD_SYSTEM_ROOT_IMAGE ?= true
endif
# ============================
ifeq ($(TARGET_USE_DYNAMIC_PARTITIONS),true)
  BOARD_SUPER_PARTITION_SIZE := 6442450944
  BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions
  BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product
  BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 6442450944
  BOARD_SUPER_PARTITION_METADATA_DEVICE := vda
  BOARD_BUILD_SUPER_IMAGE_BY_DEFAULT := true
  BOARD_SUPER_IMAGE_IN_UPDATE_PACKAGE := true
  TARGET_RELEASETOOLS_EXTENSIONS := device/google/cuttlefish/shared
else
  # ...
endif
```

**这里是模拟cuttlefish原生动态分区的配置，重点如下:**

+ 不带`PRODUCT_RETROFIT_DYNAMIC_PARTITIONS`, 原生动态分区
+ super分区大小为6442450944，即6G
+ 定义了一个动态分区组google_dynamic_partitions, 大小为6442450944, 包含三个子分区system vendor product
+ 指定了metadata数据存放的分区vda
+ `BOARD_BUILD_SUPER_IMAGE_BY_DEFAULT := true`指定了super.img由`$(PRODUCT_OUT)`目录下的文件创建，并输出到`$(PRODUCT_OUT)/super.img`中

***

# 3. 动态分区参数检查

**设置了动态分区参数以后，Android 在编译时会对参数进行检查，检查的内容包括两类：**

+ 开关参数检查，检查动态分区的配置开关是否冲突
+ 分区大小参数的检查，检查分区大小设置是否符合要求

## 3.1. 开关参数检查

> 文件`build/make/core/config.mk`的811~878行，对动态分区的开关参数进行检查

**检查重点：**

1. 改造动态分区开关和动态分区总开关必须同时设置

```shell
# 总开关
PRODUCT_USE_DYNAMIC_PARTITIONS := true
# 改造(retrofit)动态分区开关
PRODUCT_RETROFIT_DYNAMIC_PARTITIONS := true
```

2. 打开了动态分区之后，列表(system, vendor, odm, product, product_services)对应分区的以下SIZE配置不能同时设置

```shell
# (system, vendor, odm, product, product_services)
BOARD_$(device)IMAGE_PARTITION_SIZE
BOARD_$(device)IMAGE_PARTITION_RESERVED_SIZE
```

3. 对每一个分组group，需要同时设置PARTITION_LIST和SIZE参数

```shell
BOARD_$(group)_PARTITION_LIST
BOARD_$(group)_SIZE
```

4. 如果分组没有设置`BOARD_$(group)_PARTITION_LIST`, 则默认分组内没有分区
5. 分组名`BOARD_SUPER_PARTITION_GROUPS`不能设置为列表`(system vendor product product_services odm)`中的名字
6. 打开动态分区后，不需要再设置`BOARD_BUILD_SYSTEM_ROOT_IMAGE = true`

***

## 3.2. 分区大小限制

> 文件`build/make/core/Makefile`的3375~3485行，定义了多个宏对动态分区以及子分区的大小进行检查，主要是各分区或分组大小数值的计算和比较

**Google官网的说明：**

```shell
对于虚拟 A/B 启动设备，所有组的最大大小总和不得超过：
BOARD_SUPER_PARTITION_SIZE - 开销

对于 A/B 启动设备，所有组的最大大小总和必须为：
BOARD_SUPER_PARTITION_SIZE/ 2 - 开销

对于非 A/B 设备和改造的 A/B 设备，所有组的大小上限总和必须为：
BOARD_SUPER_PARTITION_SIZE - 开销

在构建时，更新组中每个分区的映像大小总和不得超过组的大小上限。
在计算时需要扣除开销，因为要考虑元数据、对齐等。合理的开销是 4 MiB，但您可以根据设备的需要选择更大的开销。
```

**上面提到的 4M 总开销的来源，主要有两类：**

1. 元数据(metadata)开销，元数据位于分区开始的 4KB~1MB 范围内
2. 分区对齐开销，默认分区按照1MB对齐

**如果动态分区中定义了一个分区组，包含三个分区(system, vendor, product)，对于A/B系统，分区组会有两个槽位，因此一共有6个子分区。**

**按中值计算，平均每个子分区对齐开销为0.5M，这样6个分区对齐，一共需要`0.5M x 6 = 3M`的总对齐开销。再加上元数据(metadata) 1M的开销，所以预估`4M = 1M + 0.5M x 6`的总开销是合理的**

***

# 4. 动态分区参数结果查看

在build/make/core/Makefile中定义了一个宏函数`dump-dynamic-partitions-info`，用于将原生动态分区相关信息输出到指定的文件中，如下:

```shell
# $(1): file
define dump-dynamic-partitions-info
  $(if $(filter true,$(PRODUCT_USE_DYNAMIC_PARTITIONS)), \
    echo "use_dynamic_partitions=true" >> $(1))
  $(if $(filter true,$(PRODUCT_RETROFIT_DYNAMIC_PARTITIONS)), \
    echo "dynamic_partition_retrofit=true" >> $(1))
  echo "lpmake=$(notdir $(LPMAKE))" >> $(1)
  $(if $(filter true,$(PRODUCT_BUILD_SUPER_PARTITION)), $(if $(BOARD_SUPER_PARTITION_SIZE), \
    echo "build_super_partition=true" >> $(1)))
  $(if $(filter true,$(BOARD_BUILD_RETROFIT_DYNAMIC_PARTITIONS_OTA_PACKAGE)), \
    echo "build_retrofit_dynamic_partitions_ota_package=true" >> $(1))
  echo "super_metadata_device=$(BOARD_SUPER_PARTITION_METADATA_DEVICE)" >> $(1)
  $(if $(BOARD_SUPER_PARTITION_BLOCK_DEVICES), \
    echo "super_block_devices=$(BOARD_SUPER_PARTITION_BLOCK_DEVICES)" >> $(1))
  $(foreach device,$(BOARD_SUPER_PARTITION_BLOCK_DEVICES), \
    echo "super_$(device)_device_size=$(BOARD_SUPER_PARTITION_$(call to-upper,$(device))_DEVICE_SIZE)" >> $(1);)
  $(if $(BOARD_SUPER_PARTITION_PARTITION_LIST), \
    echo "dynamic_partition_list=$(BOARD_SUPER_PARTITION_PARTITION_LIST)" >> $(1))
  $(if $(BOARD_SUPER_PARTITION_GROUPS),
    echo "super_partition_groups=$(BOARD_SUPER_PARTITION_GROUPS)" >> $(1))
  $(foreach group,$(BOARD_SUPER_PARTITION_GROUPS), \
    echo "super_$(group)_group_size=$(BOARD_$(call to-upper,$(group))_SIZE)" >> $(1); \
    $(if $(BOARD_$(call to-upper,$(group))_PARTITION_LIST), \
      echo "super_$(group)_partition_list=$(BOARD_$(call to-upper,$(group))_PARTITION_LIST)" >> $(1);))
  $(if $(filter true,$(TARGET_USERIMAGES_SPARSE_EXT_DISABLED)), \
    echo "build_non_sparse_super_partition=true" >> $(1))
  $(if $(filter true,$(BOARD_SUPER_IMAGE_IN_UPDATE_PACKAGE)), \
    echo "super_image_in_update_package=true" >> $(1))
endef
```

**调用`dump-dynamic-partitions-info`主要有以下3个地方:**

1. 生成`BUILT_TARGET_FILES_PACKAGE`目标时，将动态分区信息追加到`$(zip_root)/META/misc_info.txt`文件中
2. 被宏`dump-super-image-info`内部调用，在编译`super.img`或`super_empty.img`时，将动态分区信息输出到各自对应的两处`misc_info.txt`中

（1）例如，下面是一个Broadcom某平台上super.img的`misc_info.txt`内容:

```shell
$ cat out/target/product/inuvik/obj/PACKAGING/superimage_debug_intermediates/misc_info.txt
use_dynamic_partitions=true
lpmake=lpmake
build_super_partition=true
super_metadata_device=super
super_block_devices=super
super_super_device_size=3028287488
dynamic_partition_list= system vendor
super_partition_groups=bcm_ref
super_bcm_ref_group_size=1509949440
super_bcm_ref_partition_list=system vendor
ab_update=true
system_image=out/target/product/inuvik/system.img
vendor_image=out/target/product/inuvik/vendor.img
```

（2）下面是谷歌crosshatch设备super_empty.img 文件misc_info.txt的内容:

```shell
$ cat out/target/product/crosshatch/obj/PACKAGING/super_empty_intermediates/misc_info.txt
use_dynamic_partitions=true
dynamic_partition_retrofit=true
lpmake=lpmake
build_super_partition=true
build_retrofit_dynamic_partitions_ota_package=true
super_metadata_device=system
super_block_devices=system vendor product
super_system_device_size=2952790016
super_vendor_device_size=805306368
super_product_device_size=314572800
dynamic_partition_list= system vendor product
super_partition_groups=google_dynamic_partitions
super_google_dynamic_partitions_group_size=4069523456
super_google_dynamic_partitions_partition_list=system vendor product
ab_update=true
```

***

# 5. 原生动态分区super.img的生成

阅读`build/make/core/Makefile`，有两个地方去生成super.img， 一个地方生成super_empty.img, 在生成这些文件时通过脚本`build_super_image.py`调用`lpmake`去生成metadata，所以总共调用了3次:

1. 目标: superimage_dist，注释: super partition image(dist)（代码: http://aospxref.com/android-10.0.0_r47/xref/build/make/core/Makefile#4423）

dist模式下基于target_files (例如:inuvik-target_files-eng.rg935739.zip) 的内容生成super.img，其生成的文件位于:`out/target/product/inuvik/obj/PACKAGING/super.img_intermediates/super.img`

主要由`superimage_dist`目标构成依赖关系路径:`dist --> dist_files --> superimage_dist --> super.img`

2. 目标: superimage, 注释: super partition image for development（代码: http://aospxref.com/android-10.0.0_r47/xref/build/make/core/Makefile#4460）

debug模式下基于misc_info.txt的内容生成super.img，其生成的文件位于:`out/target/product/inuvik/super.img`

主要由`superimage`目标构成依赖关系路径:`droid --> droidcore --> superimage --> super.img`

**注意：相对于release而言，这里编译出来的镜像是开发时使用的，而通过`make dist`得到的镜像，是release使用的**

3. 目标: superimage_empty, 注释: super empty image（代码: http://aospxref.com/android-10.0.0_r47/xref/build/make/core/Makefile#4514）

基于misc_info.txt文件生成的super_empty.img，其生成的文件位于:`out/target/product/inuvik/super_empty.img`

主要通过`main.mk`中的`superimage_empty`目标形成依赖关系路径:`dist --> dist_files --> superimage_dist --> super_empty.img`

***

# 6. 小结

1. 动态分区参数有两类设置，一类是原生动态分区配置，一类是改造动态分区配置
2. 动态分区虽然有两套参数，但最终这两套参数会合二为一成为同一套参数，并将这些参数设置输出到misc_info.txt中。两套参数的处理细节请参考文件Android Q源码`build/make/core/config.mk`的923~994行
3. 编译系统调用build_super_image.py脚本读取`misc_info.txt`中的动态分区配置参数，传递给lpmake工具。lpmake根据动态分区参数中各分区的大小以及image路径，生成最终的`super.img`(包括metadata和各分区image)
4. 默认生成的super.img只包含了slot a的镜像，另外一个slot b为空，可以使用`lpdump`分析metadata，或者使用`lpunpack`解包查看

***

# 7. 参考

+ [Android AOSP源码](http://aospxref.com/)
+ [Android 动态分区详解(三) 动态分区配置及super.img的生成](https://blog.csdn.net/guyongqiangx/article/details/124052932)
+ [Android10 动态分区介绍](https://blog.csdn.net/u012932409/article/details/105075851)