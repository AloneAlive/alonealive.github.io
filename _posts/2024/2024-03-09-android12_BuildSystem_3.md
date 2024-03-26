---
layout: single
related: false
title:  Android 12编译系统-3 打包image镜像流程
date:   2024-03-09 18:50:02 +0800
categories: android
tags: android build
toc: true
---

> 在Android 12 AOSP源码的`build/core/main.mk`中定义了很多伪目标，我们可以直接通过`make 目标名称`进行编译，镜像的生成定义也在该文件中。本篇主要以system镜像为例，进行流程梳理分析。

# 1. build/core/main.mk系统镜像定义

这些镜像有：

伪目标名称|说明
:-|:-
.PHONY: ramdisk                     |执行`$(INSTALLED_RAMDISK_TARGET)`，生成ramdisk.img镜像
.PHONY: ramdisk_debug               |执行`$(INSTALLED_DEBUG_RAMDISK_TARGET)`，生成ramdisk-debug.img镜像
.PHONY: ramdisk_test_harness        |执行`$(INSTALLED_TEST_HARNESS_RAMDISK_TARGET)`，生成ramdisk-test-harness.img镜像
.PHONY: userdataimage               |执行`$(INSTALLED_USERDATAIMAGE_TARGET)`，生成userdata.img镜像
.PHONY: cacheimage                  |执行`$(INSTALLED_CACHEIMAGE_TARGET)`，生成cache.img（但是A/B系统不包含recovery.img和cache.img，即Android O及之后）
.PHONY: vendorimage                 |执行`$(INSTALLED_VENDORIMAGE_TARGET)`，生成vendor.img镜像
.PHONY: vendorbootimage             |执行`$(INSTALLED_VENDOR_BOOTIMAGE_TARGET)`，生成vendor_boot.img镜像
.PHONY: vendorbootimage_debug       |执行`$(INSTALLED_VENDOR_DEBUG_BOOTIMAGE_TARGET)`，生成vendor_boot-debug.img镜像
.PHONY: vendorbootimage_test_harness|执行`$(INSTALLED_VENDOR_TEST_HARNESS_BOOTIMAGE_TARGET)`，生成vendor_boot-test-harness.img镜像
.PHONY: systemextimage              |执行`$(INSTALLED_SYSTEM_EXTIMAGE_TARGET)`，生成system_ext.img镜像
.PHONY: superimage_empty            |执行`$(INSTALLED_SUPERIMAGE_EMPTY_TARGET)`，生成super_empty.img镜像
.PHONY: bootimage                   |执行`$(INSTALLED_BOOTIMAGE_TARGET)`，生成boot.img镜像
.PHONY: vbmetaimage                 |执行`$(INSTALLED_VBMETAIMAGE_TARGET)`，生成vbmeta.img镜像
.PHONY: droidcore-unbundled         |执行完整系统构建所需的目标子集，包含上面一系列的` $(INSTALLED_***)`，**其中重要的是定义了`$(INSTALLED_SYSTEMIMAGE_TARGET)`，生成system.img**
.PHONY: droidcore                   |包含上面的droidcore-unbundled，构建一系列镜像

在该文件的目标`droidcore-unbundled`中，定义了`$(INSTALLED_SYSTEMIMAGE_TARGET)`，该宏会生成system.img

***

# 2. 打包system镜像

而真正的打包实现是在`build/core/Makefile`中，make伪目标定义如下，所以我们可以通过`make systemimage`编译打包system镜像。

```makefile
# 表示在打包system.img之前，要根据依赖规则重新生成所有要进行打包的文件
.PHONY: systemimage
systemimage:

systemimage: $(INSTALLED_SYSTEMIMAGE_TARGET)

# 不需要根据依赖规则重新生成所有需要打包的文件而直接打包system.img文件
.PHONY: systemimage-nodeps snod
systemimage-nodeps snod: $(filter-out systemimage-nodeps snod,$(MAKECMDGOALS)) \
	            | $(INTERNAL_USERIMAGES_DEPS)
	@echo "make $@: ignoring dependencies"
	$(call build-systemimage-target,$(INSTALLED_SYSTEMIMAGE_TARGET))
	$(hide) $(call assert-max-image-size,$(INSTALLED_SYSTEMIMAGE_TARGET),$(BOARD_SYSTEMIMAGE_PARTITION_SIZE))
```

从上面截取的代码看，systemimage依赖于`$(INSTALLED_SYSTEMIMAGE_TARGET)`

***

## 2.1. make systemimage流程

在Android 12 AOSP源码中，执行`make systemimage`编译打包system.img，结合日志和代码，流程大致如下：

1. build.go->dumpvars.go - Build()函数执行`runMakeProductConfig`：主要配置编译参数、环境变量，同时打印到终端
2. build.go->soong.go - Build()函数执行`runSoong`：对工具进行编译，编译出blueprint等编译工具, 把`*.bp`编译成`out/soong/build.ninja`
3. build.go->kati.go - Build()函数执行`runKatiBuild`：加载`build/make/core/main.mk`，搜集所有的Android.mk文件生成ninja文件：`out/build-aosp_arm.ninja`，执行Makefile的`systemimage`目标Target
4. build.go - createCombinedBuildNinjaFile：将`out/soong/build.ninja` 、`out/build-aosp_arm.ninja`和`out/build-aosp_arm-package.ninja`， 合成为`out/out/combined-AOSP.ninja`
5. build.go->ninja.go - Build()函数执行`runNinjaForBuild`：使用ninja进行完整的编译过程，此处会执行`build/make/core/Makefile`中system镜像打包和拷贝的逻辑

### 2.1.1. make systemimage编译日志

```log
13:24:19 PROPRIETARY_HAS_SOURCECODE false
MyDebugLog ---- Not Use Bazel!
// 执行RunProductConfig
MyDebugLog ****************9 RunProductConfig!
.....
============================================
// 执行runSoong
MyDebugLog ------------------- runSoong!
// --- 执行bootstrapBlueprint
MyDebugLog runsoong****************bootstrapBlueprint Start
MyDebugLog runsoong****************bootstrapBlueprint ---> call bootstrap.RunBlueprint Start
MyDebugLog runsoong****************bootstrapBlueprint ---> call bootstrap.RunBlueprint END
// --- 构建bpglob
MyDebugLog runsoong****************Build bpglob ....
MyDebugLog runsoong****************Command ....[  2% 1/48] regenerate globs shard 189 of 1024
........
Clang SA is not enabled
// 执行runsoong结束
MyDebugLog runsoong****************END !!! ....No need to regenerate ninja file
// 执行runKatiBuild
MyDebugLog ------------------- runKatiBuild!
MyDebugLog runKatiBuild****************START !!! ....
MyDebugLog runKatiBuild****************CALL runKati START ....build/make/core/Makefile was modified, regenerating...
//加载main.mk
build/make/core/main.mk:7: warning: *******************MyDebugLog*****************Call main.mk START......
[100% 49/49] initializing build system ...
......
[ 99% 424/426] including out/soong/late-<product_name>.mk ...
//代码位置：build/core/main.mk
//$(info [$(call inc_and_print,subdir_makefiles_inc)/$(subdir_makefiles_total)] finishing build rules ...)
[ 99% 425/426] finishing build rules ...
//执行Makefile的make systemimage
build/make/core/Makefile:703: warning: *******************MyDebugLog*****************make systemimage START......
build/make/core/Makefile:1286: warning: *******************MyDebugLog*****************PRODUCT_NOTICE_SPLIT is true START......
build/make/core/Makefile:2736: warning: *******************MyDebugLog*****************INTERNAL_SYSTEMIMAGE_FILES START......
build/make/core/Makefile:2770: warning: *******************MyDebugLog*****************if BUILDING_SYSTEM_IMAGE is true START......
build/make/core/Makefile:2773: warning: *******************MyDebugLog*****************if BUILDING_SYSTEM_IMAGE is true INNER START......
build/make/core/Makefile:2825: warning: *******************MyDebugLog*****************BUILT_SYSTEMIMAGE to  build-systemimage-target......
.....
//执行runKati结束
MyDebugLog runKatiBuild****************CALL runKati END ....
MyDebugLog runKatiBuild****************CALL distGzipFile END ....
MyDebugLog runKatiBuild****************END !!! ..No need to regenerate ninja file
//执行createCombinedBuildNinjaFile
MyDebugLogw074 ------------------- createCombinedBuildNinjaFile!
//执行distGzipFile
MyDebugLog ------------------- distGzipFile!
//执行runNinjaForBuild
MyDebugLog ------------------- runNinjaForBuild!
Starting ninja...
[ 97% 427/436] build out/target/product/<product_name>/obj/NOTICE_PRODUCT.txt
[ 98% 428/436] build out/target/product/<product_name>/obj/NOTICE_PRODUCT.xml.gz
[ 98% 429/436] build out/target/product/<product_name>/system/product/etc/NOTICE.xml.gz
[ 98% 430/436] build out/target/product/<product_name>/obj/NOTICE.txt
[ 98% 431/436] build out/target/product/<product_name>/obj/NOTICE.xml.gz
[ 99% 432/436] build out/target/product/<product_name>/system/etc/NOTICE.xml.gz
[ 99% 433/436] build out/target/product/<product_name>/system/etc/linker.config.pb
[ 99% 434/436] Installed file list: out/target/product/<product_name>/installed-files.txt
// --- 创建target/product/<product_name>/obj/PACKAGING/systemimage_intermediates目录，存放system.img
[ 99% 435/436] Target system fs image: out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates/system.img

// --- 把system.img文件从out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates拷贝到该目录out/target/product/<product_name>中
[100% 436/436] Install system fs image: out/target/product/<product_name>/system.img
MyDebugLog ***** Starting ninja....... END

#### build completed successfully (07:24 (mm:ss)) ####
```

***

### 2.1.2. Makefile执行流程图

> `make systemimage`的流程和其他镜像都定义在`build/core/main.mk`不同，而是在`build/core/Makefile`中定义的target目标，主要是通过`$(INSTALLED_SYSTEMIMAGE_TARGET)`实现镜像打包。

Makefile中的流程如下：

![](../../assets/post/2024/2024-03-09-android12_BuildSystem_3/system镜像打包.png)

***

## 2.2. $(INSTALLED_SYSTEMIMAGE_TARGET)实现

`INSTALLED_SYSTEMIMAGE_TARGET`的处理实现代码：

```makefile
# 最终生成的目标
INSTALLED_SYSTEMIMAGE_TARGET := $(PRODUCT_OUT)/system.img
SYSTEMIMAGE_SOURCE_DIR := $(TARGET_OUT)

# INSTALLED_SYSTEMIMAGE_TARGE在此处被命名为INSTALLED_StYSTEMIMAGE
# 目的是为向后兼容性创建一个别名，以防特定于设备的Makefile仍然引用旧名称
INSTALLED_SYSTEMIMAGE := $(INSTALLED_SYSTEMIMAGE_TARGET)

# INSTALLED_SYSTEMIMAGE_TARGET依赖于BUILT_SYSTEMIMAGE
$(INSTALLED_SYSTEMIMAGE_TARGET): $(BUILT_SYSTEMIMAGE)
    # 此处$@的值是：out/target/product/<product_name>/system.img
	@echo "Install system fs image: $@"
    # 然后再调用了函数copy-file-to-target进行文件拷贝
	$(copy-file-to-target)
	$(hide) $(call assert-max-image-size,$@,$(BOARD_SYSTEMIMAGE_PARTITION_SIZE))

systemimage: $(INSTALLED_SYSTEMIMAGE_TARGET)
```

**按步骤顺序说明INSTALLED_SYSTEMIMAGE_TARGET的工作：**
1. `BUILT_SYSTEMIMAGE`：生成`out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates/system.img`镜像
2. 拷贝函数`copy-file-to-target`：该函数定义在`build/make/core/definitions.mk`，主要用于是拷贝文件，并且在拷贝的过程中会保留文件的权限和覆盖已有的文件。**它会首次创建`/out/target/product/<product_name>`目录，然后把文件拷贝到该目录中**

其中`BOARD_SYSTEMIMAGE_PARTITION_SIZE`：即system分区的大小定义（一般在指定product的BoardConfig.mk）

***

### 2.2.1. Step 1：生成systemimage_intermediates/system.img

> `BUILT_SYSTEMIMAGE`最终会把system.img编译到`out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates/system.img`中。

`BUILT_SYSTEMIMAGE`依赖于`FULL_SYSTEMIMAGE_DEPS`、`INSTALLED_FILES_FILE`，然后通过调用函数`build-systemimage-target`来编译`systemimage`

```makefile
# 目录路径out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates
systemimage_intermediates := \
    $(call intermediates-dir-for,PACKAGING,systemimage)
# BUILT_SYSTEMIMAGE最终结果是生成out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates/system.img镜像
BUILT_SYSTEMIMAGE := $(systemimage_intermediates)/system.img

# $(1): output file
define build-systemimage-target
  # 此处$(1)值：out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates/system.img
  @echo "Target system fs image: $(1)"
  # 1. 创建target/product/<product_name>/obj/PACKAGING/systemimage_intermediates目录并删除这个目录下的system_image_info.txt文件
  @mkdir -p $(dir $(1)) $(systemimage_intermediates) && rm -rf $(systemimage_intermediates)/system_image_info.txt

  # 2. generate-userimage-prop-dictionary作用：生成system.img的信息，保存到system_image_info.txt中
  $(call generate-image-prop-dictionary, $(systemimage_intermediates)/system_image_info.txt,system, \
      skip_fsck=true)
  # 3. BUILD_IMAGE即out/host/linux-x86/bin/build_image可执行文件（build/make/tools/releasetools/build_image.py）
  # 生成system.img镜像文件
  PATH=$(INTERNAL_USERIMAGES_BINARY_PATHS):$$PATH \
      $(BUILD_IMAGE) \
          $(TARGET_OUT) $(systemimage_intermediates)/system_image_info.txt $(1) $(TARGET_OUT) \
          || ( mkdir -p $${DIST_DIR}; \
               cp $(INSTALLED_FILES_FILE) $${DIST_DIR}/installed-files-rescued.txt; \
               exit 1 )
endef

# 如果AB系统，则赋值BOARD_AVB_SYSTEM_KEY_PATH（一般Android O及之后均执行）
ifeq ($(BOARD_AVB_ENABLE),true)
# BOARD_AVB_SYSTEM_KEY_PATH在Android 12默认key是external/avb/test/data/testkey_rsa2048.pem
$(BUILT_SYSTEMIMAGE): $(BOARD_AVB_SYSTEM_KEY_PATH)
endif

# INSTALLED_FILES_FILE即$(PRODUCT_OUT)/installed-files.txt文件
$(BUILT_SYSTEMIMAGE): $(FULL_SYSTEMIMAGE_DEPS) $(INSTALLED_FILES_FILE)
	$(call build-systemimage-target,$@)
```

***

#### 2.2.1.1. $(FULL_SYSTEMIMAGE_DEPS)说明

```makefile
# step 1：INTERNAL_SYSTEMIMAGE_FILES定义赋值，赋值的包含：
#   (1)TARGET_OUT：out/target/product/<product_name>/system/（build/make/core/envsetup.mk），其实就是system/目录
#   (2)ALL_GENERATED_SOURCES：某些工具生成的所有文件的完整路径，要拷贝到目标设备上去的由工具自动生成的源代码文件（build/make/core/binary.mk）
#   (3)ALL_DEFAULT_INSTALLED_MODULES：应添加到已安装目标的“make-droid”集合中的目标的完整路径（要安装要目标设备上的所有的模块文件）
INTERNAL_SYSTEMIMAGE_FILES := $(sort $(filter $(TARGET_OUT)/%, \
    $(ALL_GENERATED_SOURCES) \ 
    $(ALL_DEFAULT_INSTALLED_MODULES)))

# (4)如果BOARD_USES_VENDORIMAGE为true，则/system/vendor创建软链接/vendor，作为vendor.img
ifdef BOARD_USES_VENDORIMAGE
  INTERNAL_SYSTEMIMAGE_FILES += $(call create-partition-compat-symlink,$(TARGET_OUT)/vendor,/vendor,vendor.img)
endif
# (5)如果BOARD_USES_PRODUCTIMAGE为true，则/system/product创建软链接/product，作为product.img
ifdef BOARD_USES_PRODUCTIMAGE
  INTERNAL_SYSTEMIMAGE_FILES += $(call create-partition-compat-symlink,$(TARGET_OUT)/product,/product,product.img)
endif
# (6)如果BOARD_USES_SYSTEM_EXTIMAGE为true，则/system/system_ext创建软链接/system_ext，作为system_ext.img
ifdef BOARD_USES_SYSTEM_EXTIMAGE
  INTERNAL_SYSTEMIMAGE_FILES += $(call create-partition-compat-symlink,$(TARGET_OUT)/system_ext,/system_ext,system_ext.img)
endif

# step 2：赋值给FULL_SYSTEMIMAGE_DEPS
#   INTERNAL_SYSTEMIMAGE_FILES：就是上面得到的最终值
#   INTERNAL_USERIMAGES_DEPS：制作system.img镜像所依赖的工具
FULL_SYSTEMIMAGE_DEPS := $(INTERNAL_SYSTEMIMAGE_FILES) $(INTERNAL_USERIMAGES_DEPS)

# 系统映像中的ASAN库-添加依赖项
ASAN_IN_SYSTEM_INSTALLED := $(TARGET_OUT)/asan.tar.bz2
ifneq (,$(filter address, $(SANITIZE_TARGET)))
  ifeq (true,$(SANITIZE_TARGET_SYSTEM))
    FULL_SYSTEMIMAGE_DEPS += $(ASAN_IN_SYSTEM_INSTALLED)
  endif
endif

# $(INTERNAL_ROOT_FILES)：目录out/target/product/<product_name>/root/
# $(INSTALLED_FILES_FILE_ROOT)：文件out/target/product/<product_name>/installed-files-root.txt
FULL_SYSTEMIMAGE_DEPS += $(INTERNAL_ROOT_FILES) $(INSTALLED_FILES_FILE_ROOT)
```

***

#### 2.2.1.2. $(INSTALLED_FILES_FILE)说明

`INSTALLED_FILES_FILE`依赖的是文件`$(PRODUCT_OUT)/installed-files.txt`，这是已安装的文件列表，这些文件要打包到system.img中，他也依赖于`FULL_SYSTEMIMAGE_DEPS`

```makefile
# installed file list已安装列表
# 取决于$（BUILT_SYSTEMIMAGE）所依赖的任何内容。
# 在依赖关系图中，我们将installed-files.txt放在映像本身之前，这样即使由于系统映像过大而导致构建失败，我们也可以获得大小统计。
INSTALLED_FILES_FILE := $(PRODUCT_OUT)/installed-files.txt
INSTALLED_FILES_JSON := $(INSTALLED_FILES_FILE:.txt=.json)
$(INSTALLED_FILES_FILE): .KATI_IMPLICIT_OUTPUTS := $(INSTALLED_FILES_JSON)
$(INSTALLED_FILES_FILE): $(FULL_SYSTEMIMAGE_DEPS) $(FILESLIST) $(FILESLIST_UTIL)
	@echo Installed file list: $@
	mkdir -p $(dir $@)
	rm -f $@
	$(FILESLIST) $(TARGET_OUT) > $(@:.txt=.json)
	$(FILESLIST_UTIL) -c $(@:.txt=.json) > $@

.PHONY: installed-file-list
installed-file-list: $(INSTALLED_FILES_FILE)

$(call dist-for-goals, sdk win_sdk sdk_addon, $(INSTALLED_FILES_FILE))
```

***

### 2.2.2. Step 2：copy-file-to-target拷贝

拷贝函数`copy-file-to-target`：该函数定义在`build/make/core/definitions.mk`，主要用于是拷贝文件，并且在拷贝的过程中会保留文件的权限和覆盖已有的文件。**它会创建`/out/target/product/<product_name>`目录，然后把system.img文件从`out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates`拷贝到该目录`out/target/product/<product_name>`中**

```makefile
# 将单个文件从一个位置复制到另一个位置，
# 保留权限并覆盖任何现有的文件
define copy-file-to-target
@mkdir -p $(dir $@)
$(hide) rm -f $@
$(hide) cp "$<" "$@"
endef
```

通过对比这两个目录的system.img文件的md5值，可以发现是一样的。

```shell
android/out/target/product/<product_name>/obj/PACKAGING/systemimage_intermediates$ md5sum system.img 
c87d3ca1aa1462b2d97102dedf6fca28  system.img

android/out/target/product/<product_name>$ md5sum system.img 
c87d3ca1aa1462b2d97102dedf6fca28  system.img
```

***

# 3. 参考

+ [Image打包流程-Android10.0编译系统（四）](https://mp.weixin.qq.com/s?__biz=MjM5NDk5ODQwNA==&mid=2652469367&idx=1&sn=820ce34bf318976777fb7e75ccb92013&chksm=bd12e99c8a65608ad003c9f35b36152966e037a4e5a441fe12ced5c93cee24a8be2125eb06d2&scene=178&cur_album_id=1552818877418487808#rd)
+ [Android编译系统分析五：system.img的生成过程](https://blog.csdn.net/u011913612/article/details/52503318)
