---
layout: single
title:  AB升级 升级包生成制作流程和常见问题现象小结
date:   2022-04-26 14:19:02 +0800 
categories: ota 
tags: android ota AB升级
toc: true
---

> 升级包生成文件、升级方式、常见问题分析调试方法、make otapackage升级包脚本流程解析

# 1. 升级包生成方式

Android升级包使用`make otapackage`打包生成，会生成target压缩包（包含完整的image数据）和可用于升级的ota update压缩包。

升级包包含全量包和差分包，差分包的制作需要两个不同版本的target包，然后使用`./build/tools/releasetools/ota_from_target_files –i A-target_files.zip B-target_files.zip incremental_ota_update.zip`脚本命令制作

# 2. 升级包目录

**升级包解压后可以查看文件目录：**

```shell
├── META-INF
│   └── com
│       └── android
│           ├── metadata  //升级包版本信息
│           └── otacert
├── payload.bin           //升级包数据，可用工具解析获取升级的image数据
└── payload_properties.txt  //包含FILE_HASH、FILE_SIZE、METADATA_HASH、METADATA_SIZE四个文件元信息
```

**target包解压后：**

```shell
target$ tree -L 1
.
├── BOOT
├── IMAGES
├── META
├── OTA
├── PREBUILT_IMAGES
├── RADIO
├── ROOT
├── SYSTEM
└── VENDOR
```

## 2.1. 升级脚本和方法

```shell
android/system/update_engine/scripts$ tree
.
├── blockdiff.py
├── brillo_update_payload
├── paycheck.py
├── payload_info.py
├── payload_info_unittest.py
├── run_unittests
├── test_paycheck.sh
├── update_device.py
└── update_payload
```

当AB系统升级时，有两种方式来调用updateengine,来实现升级：
（1） 一种方法是直接执行shell命令，调用update_engine_client，带参数来实现升级

```shell
//解压升级包
adb shell
update_engine_client --payload=file:///storage/5F49-FB9D/socupdate8g/payload.bin --update --headers="FILE_HASH=YP7Z1bFDv6O8C5LTWZ20JxTljXyoVitlCX27TBTyVDM=
FILE_SIZE=967460335
METADATA_HASH=1gpTz/Q7T1ysTu6suP8N2KVOfa+vKEdnJGnPsKcPiXw=
METADATA_SIZE=75378"
```

（2） 另一种方式是应用层直接调用UpdateEngine的applyPayload方法来升级


调试方式打印日志：`adb logcat -s update_engine`

通过`cat proc/cmdline`查看升级前后的AB分区切换，判断升级是否成功

***

# 3. 常见错误现象分析

## 3.1. 重复升级同版本报错

（1）删除`/data/misc/update_engine/prefs`目录下记录的信息文件

```shell
/data/misc/update_engine/prefs # ls -al
total 60
drwx------ 2 root root 4096 1970-01-01 08:00 .
drwx------ 3 root root 4096 1970-01-01 08:00 ..
-rw------- 1 root root   36 1970-01-01 08:00 boot-id
-rw------- 1 root root    1 2021-12-12 08:15 delta-update-failures
-rw------- 1 root root    2 1970-01-01 08:00 manifest-metadata-size
-rw------- 1 root root    2 1970-01-01 08:00 manifest-signature-size
-rw------- 1 root root   24 1970-01-01 08:00 previous-version
-rw------- 1 root root    1 1970-01-01 08:00 resumed-update-failures
-rw------- 1 root root    1 2021-12-12 08:15 total-bytes-downloaded
-rw------- 1 root root   88 2021-12-12 08:06 update-check-response-hash
-rw------- 1 root root   36 2021-12-12 08:15 update-completed-on-boot-id
-rw------- 1 root root    1 1970-01-01 08:00 update-state-next-data-length
-rw------- 1 root root    2 1970-01-01 08:00 update-state-next-data-offset
-rw------- 1 root root    2 1970-01-01 08:00 update-state-next-operation
-rw------- 1 root root    0 1970-01-01 08:00 update-state-sha-256-context
-rw------- 1 root root    0 1970-01-01 08:00 update-state-signature-blob
-rw------- 1 root root    0 1970-01-01 08:00 update-state-signed-sha-256-context
/data/misc/update_engine/prefs # rm -rf *
```

（2）修改/system/update_engine/update_attempter_android.cc文件，添加如下代码.

这样就会在升级完成后，删除`/data/misc/update_engine/prefs/update-check-response-hash`文件；再使用任意升级包升级，都会认为是一次全新的升级

```diff
diff --git a/update_attempter_android.cc b/update_attempter_android.cc
--- a/update_attempter_android.cc
+++ b/update_attempter_android.cc
@@ -458,7 +458,7 @@ void UpdateAttempterAndroid::ProcessingDone(const ActionProcessor* processor,
       // Update succeeded.
       WriteUpdateCompletedMarker();
       prefs_->SetInt64(kPrefsDeltaUpdateFailures, 0);
-
+      prefs_->Delete(kPrefsUpdateCheckResponseHash);
       LOG(INFO) << "Update successfully applied, waiting to reboot.";
       break;
```

## 3.2. 回滚版本升级报错

update engine会校验版本构建的时间戳，修改将时间戳的校验删除即可

```diff
diff --git a/payload_consumer/delta_performer.cc b/payload_consumer/delta_performer.cc
--- a/payload_consumer/delta_performer.cc
+++ b/payload_consumer/delta_performer.cc
@@ -1686,13 +1686,14 @@ ErrorCode DeltaPerformer::ValidateManifest() {
     }
   }
 
+  /* Delete for updating to old version
   if (manifest_.max_timestamp() < hardware_->GetBuildTimestamp()) {
     LOG(ERROR) << "The current OS build timestamp ("
                << hardware_->GetBuildTimestamp()
                << ") is newer than the maximum timestamp in the manifest ("
                << manifest_.max_timestamp() << ")";
     return ErrorCode::kPayloadTimestampError;
-  }
+  }*/

```

**解决方法：**

1）全量包：修改`/system/update_engine/payload_consumer/delta_performer.cc`文件，将时间戳校验的相关代码注释掉

2）差分包：差分包和全量包不同，如果想做新版本差分到旧版本的包，需要在使用ota_from_target_files.py脚本制作升级包时添加参数`—override_timestamp`，这样就可以跳过时间戳的检测

## 3.3. 差分包升级error code=20(kDownloadStateInitializationError)

> 错误码见：`system/update_engine/common/error_code.h`

（1）如果是如下log，则当前版本应该是userdebug版本，设备有被进行过remount操作，需要整包升级/线刷恢复

```log
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337259:ERROR:fec_file_descriptor.cc(30)] No ECC data in the passed file
//system b分区
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337493:ERROR:delta_performer.cc(430)] Unable to open ECC source partition system on slot B, file /dev/block/by-name/system_b: No such file or directory (2)
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337571:ERROR:delta_performer.cc(1135)] The hash of the source data on disk for this operation doesn't match the expected value. This could mean that the delta update payload was targeted for another version, or that the source partition was modified after it was installed, for example, by mounting a filesystem.
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337627:ERROR:delta_performer.cc(1140)] Expected:   sha256|hex = 8D224141934E427E63257E32134DA8B23FE280F8A2643365A6AD55701EAA8575
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337679:ERROR:delta_performer.cc(1143)] Calculated: sha256|hex = 3EEC85740D17A75F584976B29D8D62619E68257E6DC324D2206FDFE29E30DCA4
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337742:ERROR:delta_performer.cc(1154)] Operation source (offset:size) in blocks: 0:2,322:1,326:1,348:1,351:1,359:299,838:2,3044:202,3472:2,3749:1
04-01 18:33:13.337  2631  2631 W update_engine: [0401/183313.337820:WARNING:mount_history.cc(66)] Device was remounted R/W 2 times. Last remount happened on 2022-04-01 10:32:25.000 UTC.
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337900:ERROR:delta_performer.cc(1435)] source_fd != nullptr failed.
04-01 18:33:13.337  2631  2631 E update_engine: [0401/183313.337972:ERROR:delta_performer.cc(296)] Failed to perform BROTLI_BSDIFF operation 172, which is the operation 0 in partition "system"
04-01 18:33:13.338  2631  2631 E update_engine: [0401/183313.338040:ERROR:download_action.cc(336)] Error ErrorCode::kDownloadStateInitializationError (20) in DeltaPerformer's Write method when processing the received payload -- Terminating processing
04-01 18:33:13.338  2631  2631 I update_engine: [0401/183313.338527:INFO:delta_performer.cc(313)] Discarding 4265 unused downloaded bytes
04-01 18:33:13.338  2631  2631 I update_engine: [0401/183313.338670:INFO:multi_range_http_fetcher.cc(177)] Received transfer terminated.
04-01 18:33:13.338  2631  2631 I update_engine: [0401/183313.338739:INFO:multi_range_http_fetcher.cc(129)] TransferEnded w/ code 200
04-01 18:33:13.338  2631  2631 I update_engine: [0401/183313.338791:INFO:multi_range_http_fetcher.cc(131)] Terminating.
...
04-01 18:33:13.381  2631  2631 I update_engine: [0401/183313.381569:INFO:action_processor.cc(116)] ActionProcessor: finished DownloadAction with code ErrorCode::kDownloadStateInitializationError

```

（2）如果是其他问题，则检查对应分区的image，比如线刷包的image和target包是否一致，比如设备的image是否被改动（使用dd命令）

***

## 3.4. 差分包升级error code=15(kNewRootfsVerificationError)

该错误码和15一样都是分区hash校验失败，一个是在升级刚开始，一个是在文件系统校验的时候。

这个操作的磁盘上的源数据的散列与预期值不匹配。这可能意味着增量更新有效负载是针对另一个版本的，或者是在安装之后修改了源分区，例如，通过安装文件系统。

原因: **make otapackage 会对system.img重新打包 导致重新打包的system.img和out目录下的system.img Hash值不一致. 也就是线刷版本的system.img和OTA包的system.img不一致，整包升级会替换system.img**, 而差分包升级则需要保证系统内部的system.img和整包中system.img一致才能升级成功(差分包中保存了通过sha256 Hash算法计算出整包system.img的值,通过这个值来确定两个system.img一致)

**调试方法：**

**（1） dump获取车机的image文件**

（1）进入adb shell

（2）在/dev/block/by-name/查看问题分区

（3）使用`dd if=/dev/block/by-name/system of=/sdacar/a.img bs=512 count=5`命令获取到
(如果dump失败，尝试更改of路径)

**(2) 解析image文件**

（1）确认文件属性

```shell
//车机dd出来的img
$ file system_old.img 
system_old.img: Linux rev 1.0 ext4 filesystem data, UUID=4729639d-b5f2-5cc1-a120-9ac5f788683c (extents) (large files) (huge files)

//系统编译的img（需要转换解析）
G$ file system.img 
system.img: Android sparse image, version: 1.0, Total of 1310720 4096-byte output blocks in 86 input chunks.
```

（2）如果是sparse文件属性，则进行转化：(android编译环境)

`simg2img get.img result.img`

**（3）采用挂载分区的方式来解压打开img文件**

```shell
g$ file system_old.img 
system_old.img: Linux rev 1.0 ext4 filesystem data, UUID=4729639d-b5f2-5cc1-a120-9ac5f788683c (needs journal recovery) (extents) (large files) (huge files)
$ mkdir 1
$ sudo mount system_old.img 1/ -o loop
```

**（4）使用文件对比工具（比如beyond compare）对比两个image的md5值和文件目录**

***

# 4. 升级包生成流程解析

在源码下执行该命令生成update.zip包主要分成两步：
1. 根据`build/core/Makefile`执行编译生成一个target update原包（zip格式）
2. 运行一个python脚本，并以上一步准备的zip包作为输入，最终生成我们需要的升级包

## 4.1. Makefile编译生成target原包

这个原包在实际编译过程中有两个作用：
1. 用来生成OTA update升级包
2. 用来生成系统镜像

**编译脚本`build/core/Makefile`中：**

(参考otapackage/Makefile注释)

```makefile
# -----------------------------------------------------------------
# 一个zip压缩包用于映射target filesystem
# 这个压缩包可以用来做ota升级包或者filesystem image作为构建后的一个步骤
name := $(TARGET_PRODUCT)
ifeq ($(TARGET_BUILD_TYPE),debug)
  name := $(name)_debug
endif
# android/build/make/core/main.mk ---> FILE_NAME_TAG := eng.$(BUILD_USERNAME)
# target原包包名
# 例如：productName-target_files-eng.UserName
name := $(name)-target_files-$(FILE_NAME_TAG)
//进入该路径 obj/PACKAGING/target_files_intermediates
intermediates := $(call intermediates-dir-for,PACKAGING,target_files)
BUILT_TARGET_FILES_PACKAGE := $(intermediates)/$(name).zip
$(BUILT_TARGET_FILES_PACKAGE): intermediates := $(intermediates)
//target文件的目录
$(BUILT_TARGET_FILES_PACKAGE): \
	    zip_root := $(intermediates)/$(name)

# $(1): Directory to copy
# $(2): Location to copy it to
# The "ls -A" is to prevent "acp s/* d" from failing if s is empty.
define package_files-copy-root
  if [ -d "$(strip $(1))" -a "$$(ls -A $(1))" ]; then \
    mkdir -p $(2) && \
    $(ACP) -rd $(strip $(1))/* $(2); \
  fi
endef

built_ota_tools :=
.....
# 在此处添加后就会生成到target原始包中
$(BUILT_TARGET_FILES_PACKAGE): \
	    $(INSTALLED_RAMDISK_TARGET) \
	    $(INSTALLED_BOOTIMAGE_TARGET) \
	    $(INSTALLED_RECOVERYIMAGE_TARGET) \
	    $(FULL_SYSTEMIMAGE_DEPS) \
	    $(INSTALLED_USERDATAIMAGE_TARGET) \
	    $(INSTALLED_CACHEIMAGE_TARGET) \
	    $(INSTALLED_VENDORIMAGE_TARGET) \
	    $(INSTALLED_PRODUCTIMAGE_TARGET) \
	    $(INSTALLED_PRODUCT_SERVICESIMAGE_TARGET) \
	    $(INSTALLED_VBMETAIMAGE_TARGET) \
        ....
//image拷贝到target包的images路径下
ifdef BOARD_PREBUILT_VENDORIMAGE
	$(hide) mkdir -p $(zip_root)/IMAGES
	$(hide) cp $(INSTALLED_VENDORIMAGE_TARGET) $(zip_root)/IMAGES/
endif
ifdef BOARD_PREBUILT_PRODUCTIMAGE
	$(hide) mkdir -p $(zip_root)/IMAGES
	$(hide) cp $(INSTALLED_PRODUCTIMAGE_TARGET) $(zip_root)/IMAGES/
endif
.....
ifneq ($(BOARD_SUPER_PARTITION_GROUPS),)
	$(hide) echo "super_partition_groups=$(BOARD_SUPER_PARTITION_GROUPS)" > $(zip_root)/META/dynamic_partitions_info.txt
	@# Remove 'vendor' from the group partition list if the image is not available. This should only
	@# happen to AOSP targets built without vendor.img. We can't remove the partition from the
	@# BoardConfig file, as it's still needed elsewhere (e.g. when creating super_empty.img).
	$(foreach group,$(BOARD_SUPER_PARTITION_GROUPS), \
	    $(eval _group_partition_list := $(BOARD_$(call to-upper,$(group))_PARTITION_LIST)) \
	    $(if $(INSTALLED_VENDORIMAGE_TARGET),,$(eval _group_partition_list := $(filter-out vendor,$(_group_partition_list)))) \
	    echo "$(group)_size=$(BOARD_$(call to-upper,$(group))_SIZE)" >> $(zip_root)/META/dynamic_partitions_info.txt; \
	    $(if $(_group_partition_list), \
	        echo "$(group)_partition_list=$(_group_partition_list)" >> $(zip_root)/META/dynamic_partitions_info.txt;))
endif # BOARD_SUPER_PARTITION_GROUPS
	@# TODO(b/134525174): Remove `-r` after addressing the issue with recovery patch generation.

	# 调用add_img_to_target_files脚本，将img添加到target files包中
	$(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH MKBOOTIMG=$(MKBOOTIMG) \
	    build/make/tools/releasetools/add_img_to_target_files -a -r -v -p $(HOST_OUT) $(zip_root)
	@# Zip everything up, preserving symlinks and placing META/ files first to
	@# help early validation of the .zip file while uploading it.
	$(hide) find $(zip_root)/META | sort >$@.list
	$(hide) find $(zip_root) -path $(zip_root)/META -prune -o -print | sort >>$@.list
	$(hide) $(SOONG_ZIP) -d -o $@ -C $(zip_root) -l $@.list
```

## 4.2. ota_frome_target_files.py脚本

在编译过程中若生成OTA update升级包时会调用一个名为`ota_from_target_files`的python脚本，位置在/build/tools/releasetools/ota_from_target_files。

这个脚本的作用是以第一步生成的zip原始包作为输入，最终生成可用的OTA升级zip包。

### 4.2.1. Makefile

下面是Makefile引用的入口：

**如果不想在编译的时候生成升级包，可以将`TARGET_SKIP_OTA_PACKAGE`置成false**

```makefile
//android/build/core/Makefile
ifeq ($(BUILD_OS),darwin)
  build_ota_package := false
  build_otatools_package := false
else
  # set build_ota_package, and allow opt-out below
  build_ota_package := true
  ifeq ($(TARGET_SKIP_OTA_PACKAGE),true)
    build_ota_package := false
  endif
  .....
endif

....
//make otapackage制作升级包
ifeq ($(build_ota_package),true)
# -----------------------------------------------------------------
# OTA update package

# $(1): output file
# $(2): additional args
define build-ota-package-target
# 调用ota_from_target_files脚本制作升级包
PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH MKBOOTIMG=$(MKBOOTIMG) \
   build/make/tools/releasetools/ota_from_target_files -v \
   --block \
   --extracted_input_target_files $(patsubst %.zip,%,$(BUILT_TARGET_FILES_PACKAGE)) \
   -p $(HOST_OUT) \
   $(if $(OEM_OTA_CONFIG), -o $(OEM_OTA_CONFIG)) \
   $(2) \
   $(BUILT_TARGET_FILES_PACKAGE) $(1)
endef

name := $(TARGET_PRODUCT)
ifeq ($(TARGET_BUILD_TYPE),debug)
  name := $(name)_debug
endif
//升级包名称
name := $(name)-ota-$(FILE_NAME_TAG)
//升级包目录
INTERNAL_OTA_PACKAGE_TARGET := $(PRODUCT_OUT)/$(name).zip

INTERNAL_OTA_METADATA := $(PRODUCT_OUT)/ota_metadata
.....
```

***

### 4.2.2. ota_from_target_files

> Path:`android/build/make/tools/releasetools/ota_from_target_files.py`

这个脚本开始部分的帮助文档:

```shell
Usage: ota_from_target_files [flags] input_target_files output_ota_package
    -b 过时的。
    -k 签名所使用的密钥
    -i 生成增量OTA包时使用此选项。后面我们会用到这个选项来生成OTA增量包。
    -w 是否清除userdata分区
    -n 在升级时是否不检查时间戳，缺省要检查，即缺省情况下只能基于旧版本升级。
    -e 是否有额外运行的脚本
    -m 执行过程中生成脚本(updater-script)所需要的格式,目前有两种即amend和edify。对应上两种版本升级时会采用不同的解释器。缺省会同时生成两种格式的脚 本。
    -p 定义脚本用到的一些可执行文件的路径。
    -s 定义额外运行脚本的路径。
    -x 定义额外运行的脚本可能用的键值对。
    -v 执行过程中打印出执行的命令。
    -h 命令帮助
```

`/build/tools/releasetools/ota_from_target_files.py`脚本:

**主函数main是python的入口函数，从main函数开始看，大概看一下main函数里的流程：**

（1）在main函数的开头，首先将用户设定的option选项存入OPTIONS变量中，它是一个python中的类。紧接着判断有没有额外的脚本，如果有就读入到OPTIONS变量中。

（2）解压缩输入的zip包，即我们在上文生成的原始zip包。然后判断是否用到device-specific extensions（设备扩展）如果用到，随即读入到OPTIONS变量中。

（3）判断是否签名，然后判断是否有新内容的增量源，有的话就解压该增量源包放入一个临时变量中（source_zip）。自此，所有的准备工作已完毕，随即会调用该脚本中最主要的函数`WriteFullOTAPackage(input_zip,output_zip)`

```python
def main(argv):
  # 将用户设定的opthin选项存入OPTIONS变量中，它是一个python类
  def option_handler(o, a):
    if o in ("-k", "--package_key"):
      OPTIONS.package_key = a
    elif o in ("-i", "--incremental_from"):
      OPTIONS.incremental_source = a
	  .....
    # target原始包
    elif o == "--extracted_input_target_files":
      OPTIONS.extracted_input = a
	  .....
#直接从 zip 或提取的输入加载构建信息字典目录。 我们不需要解压缩整个目标文件 zip，因为它们 A/B OTA 不需要（brillo_update_payload 自行完成）。
# 加载 info dicts 时，我们不需要提供第二个参数到 common.LoadInfoDict()。 指定第二个参数允许替换一些带有实际路径的属性，例如“selinux_fc”，'ramdisk_dir'，在OTA生成期间不会使用。
  if OPTIONS.extracted_input is not None:
    OPTIONS.info_dict = common.LoadInfoDict(OPTIONS.extracted_input)
  else:
    with zipfile.ZipFile(args[0], 'r') as input_zip:
      OPTIONS.info_dict = common.LoadInfoDict(input_zip)

  logger.info("--- target info ---")
  common.DumpInfoDict(OPTIONS.info_dict)

  # Load the source build dict if applicable.
  #如果使用，加载源构建的dict
  if OPTIONS.incremental_source is not None:
    OPTIONS.target_info_dict = OPTIONS.info_dict
    with zipfile.ZipFile(OPTIONS.incremental_source, 'r') as source_zip:
      OPTIONS.source_info_dict = common.LoadInfoDict(source_zip)

    logger.info("--- source info ---")
    common.DumpInfoDict(OPTIONS.source_info_dict)
	  .....
  ab_update = OPTIONS.info_dict.get("ab_update") == "true"

#如果没有指定 package_key，则使用默认密钥对包进行签名。ab_updates上需要 package_keys，所以如果有ab_update正在创建
  if not OPTIONS.no_signing or ab_update:
    if OPTIONS.package_key is None:
      OPTIONS.package_key = OPTIONS.info_dict.get(
          "default_system_dev_certificate",
          "build/target/product/security/testkey")
    # Get signing keys
    OPTIONS.key_passwords = common.GetKeyPasswords([OPTIONS.package_key])
	....
# 解压原始包到一个中间文件夹，并将目录赋值给input_tmp
  if OPTIONS.extracted_input is not None:
    OPTIONS.input_tmp = OPTIONS.extracted_input
  else:
    logger.info("unzipping target target-files...")
    OPTIONS.input_tmp = common.UnzipTemp(args[0], UNZIP_PATTERN)
  OPTIONS.target_tmp = OPTIONS.input_tmp
  ....
  # Generate a full OTA.构建OTA全量包
  if OPTIONS.incremental_source is None:
    with zipfile.ZipFile(args[0], 'r') as input_zip:
      WriteFullOTAPackage(
          input_zip,
          output_file=args[1])

  # Generate an incremental OTA.构建OTA增量包
  else:
    logger.info("unzipping source target-files...")
    OPTIONS.source_tmp = common.UnzipTemp(
        OPTIONS.incremental_source, UNZIP_PATTERN)
    with zipfile.ZipFile(args[0], 'r') as input_zip, \
        zipfile.ZipFile(OPTIONS.incremental_source, 'r') as source_zip:
      WriteBlockIncrementalOTAPackage(
          input_zip,
          source_zip,
          output_file=args[1])

    if OPTIONS.log_diff:
      with open(OPTIONS.log_diff, 'w') as out_file:
        import target_files_diff
        target_files_diff.recursiveDiff(
            '', OPTIONS.source_tmp, OPTIONS.input_tmp, out_file)

```

（4） `WriteFullOTAPackage`函数的处理过程是先获得脚本的生成器。默认格式是edify。然后获得metadata元数据，此数据来至于Android的一些环境变量。然后获得设备配置参数比如api函数的版本。然后判断是否忽略时间戳。

```makefile
def WriteFullOTAPackage(input_zip, output_file):
  target_info = BuildInfo(OPTIONS.info_dict, OPTIONS.oem_dicts)

#我们不知道它将安装在哪个版本之上。 我们期待 API只是不会经常改变。 同样对于 fstab，它可能在目标构建
  target_api_version = target_info["recovery_api_version"]
  script = edify_generator.EdifyGenerator(target_api_version, target_info)

  if target_info.oem_props and not OPTIONS.oem_no_mount:
    target_info.WriteMountOemScript(script)
  # 获取meta数据
  metadata = GetPackageMetadata(target_info)
  ......
# 如果阶段不是“2/3”或“3/3”：将恢复映像写入启动分区，设置舞台为“2/3”，重新启动以引导分区并重新启动恢复；
# 否则如果阶段是“2/3”：将恢复映像写入恢复分区，设置舞台为“3/3”，重启到恢复分区并重启恢复
# 别的：（阶段必须是“3/3”），设置舞台为“”，进行正常的完整包安装：擦除并安装系统、启动映像等。设置系统在第一次启动时更新恢复分区，正常完成脚本（允许恢复标记自己完成并重新启动）
  recovery_img = common.GetBootableImage("recovery.img", "recovery.img",
                                         OPTIONS.input_tmp, "RECOVERY")
  if OPTIONS.two_step:
    if not target_info.get("multistage_support"):
      assert False, "two-step packages not supported by this build"
    fs = target_info["fstab"]["/misc"]
    assert fs.fs_type.upper() == "EMMC", \
        "two-step packages only supported on devices with EMMC /misc partitions"
    bcb_dev = {"bcb_dev": fs.device}
    common.ZipWriteStr(output_zip, "recovery.img", recovery_img.data)
    script.AppendExtra("""
if get_stage("%(bcb_dev)s") == "2/3" then
""" % bcb_dev)

    # Stage 2/3: Write recovery image to /recovery (currently running /boot).
    script.Comment("Stage 2/3")
    script.WriteRawImage("/recovery", "recovery.img")
    script.AppendExtra("""
set_stage("%(bcb_dev)s", "3/3");
reboot_now("%(bcb_dev)s", "recovery");
else if get_stage("%(bcb_dev)s") == "3/3" then
""" % bcb_dev)

    # Stage 3/3: Make changes.
    script.Comment("Stage 3/3")

  # Dump fingerprints
  script.Print("Target: {}".format(target_info.fingerprint))

  device_specific.FullOTA_InstallBegin()

  system_progress = 0.75

  if OPTIONS.wipe_user_data:
    system_progress -= 0.1
  if HasVendorPartition(input_zip):
    system_progress -= 0.1

  script.ShowProgress(system_progress, 0)

# 调用函数获取meta数据
def GetPackageMetadata(target_info, source_info=None):
# 生成并返回元数据字典。它生成一个 dict() ，其中包含要写入 OTA 的信息包 (META-INF/com/android/metadata)。 它还处理检测基于全局选项降级/数据擦除。
# 参数：
#     target_info：保存目标构建信息的 BuildInfo 实例。
#     source_info：保存源构建信息的 BuildInfo 实例，或
#     如果生成完整的 OTA，则无。
# 返回： 要写入包元数据条目的字典。
  assert isinstance(target_info, BuildInfo)
  assert source_info is None or isinstance(source_info, BuildInfo)
....
```

（5）`WriteFullOTAPackage`函数做完准备工作后就开始生成升级用的脚本文件(updater-script)了。生成脚本文件后将上一步获得的metadata元数据写入到输出包out_zip

（6）至此一个完整的update.zip升级包就生成了。生成位置在：`out/target/product/tcc8800/full_tcc8800_evm-ota-eng.mumu.20120315.155326.zip`。将升级包拷贝到SD卡中就可以用来升级了。

***

## 4.3. misc_info.txt

生成META目录下，可以查看到image的size、type等信息

该size数据是由`BOARD_DTBOIMG_PARTITION_SIZE`类似该宏定义，用于生成的最终ota包中img的大小（AB分区的大小）

修改于product/BoardConfig.mk文件中

***

# 5. ab_partitions.txt

ab分区的image文件列表

生成在`out/target/product/.../obj/PACKAGING/target_files_intermediates/..._target_files_eng.***/META/`路径

make otapackage后，会在build/core/add_img_to_target_files.py脚本中，从该文件中读取列表然后查找image name


# 6. 参考

+ [Android 编译如何跳过生成ota package过程](https://www.jianshu.com/p/1b3ff36f4138)
+ [Android OTA升级原理和流程分析（一）--update.zip包的制作](https://blog.csdn.net/twk121109281/article/details/90712880)
