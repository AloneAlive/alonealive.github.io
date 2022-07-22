---
layout: single
title:  Android badblock磁盘坏道检测调试
date:   2022-06-10 13:43:02 +0800 
categories: linux 
tags: android ota linux
toc: true
---

> Android升级的时候在FilesystemVerifierAction出现某分区Buffer I/O error读写失败，上报错误码1（error）。从问题现象看需要针对该分区进行磁盘坏道检测，分析是否是因为磁盘损坏导致。

# 1. 升级报错现象

Android AB升级到FilesystemVerifierAction步骤对分区文件系统进行校验，此时出现升级分区的读写错误。

1.查看日志，截取信息，看到其中`mmcblk0p36`分区报错

```log
.....
01-01 08:04:12.231     0     0 I mmcblk0 : error -110 sending stop command, original cmd response 0x900, card status 0x400900
01-01 08:04:12.248     0     0 I mmcblk0 : error -110 sending stop command, original cmd response 0x900, card status 0x400900
01-01 08:04:12.248     0     0 E mmcblk0 : error -84 transferring data, sector 8860736, nr 8, cmd response 0x900, card status 0x0
01-01 08:04:12.258     0     0 W mmcblk0 : retrying using single block read
01-01 08:04:12.275     0     0 E mmcblk0 : error -84 transferring data, sector 8860742, nr 2, cmd response 0x900, card status 0x0
01-01 08:04:12.396     0     0 E mmcblk0 : error -84 transferring data, sector 8860742, nr 2, cmd response 0x900, card status 0x0
01-01 08:04:12.416     0     0 E         : Buffer I/O error on dev mmcblk0p36, logical block 7416, async page read
06-08 15:20:37.123  2482  2482 E update_engine: [0608/152037.123885:ERROR:file_stream.cc(-1)] Domain=system, Code=EIO, Message=I/O error
06-08 15:20:37.124  2482  2482 E update_engine: [0608/152037.124113:ERROR:filesystem_verifier_action.cc(180)] Unable to schedule an asynchronous read from the stream.
06-08 15:20:37.151  2482  2482 I update_engine: [0608/152037.151148:INFO:action_processor.cc(116)] ActionProcessor: finished FilesystemVerifierAction with code ErrorCode::kError
06-08 15:20:37.151  2482  2482 I update_engine: [0608/152037.151365:INFO:action_processor.cc(121)] ActionProcessor: Aborting processing due to failure.
06-08 15:20:37.151  2482  2482 I update_engine: [0608/152037.151455:INFO:update_attempter_android.cc(455)] Processing Done.
06-08 15:20:37.151  2482  2482 I update_engine: [0608/152037.151523:INFO:dynamic_partition_control_android.cc(151)] Destroying [] from device mapper
```

2.查看报错分区，对应是vendor_a分区

```shell
console:/dev/block/by-name # ls -al
total 0
drwxr-xr-x 2 root root  900 1970-01-01 08:00 .
drwxr-xr-x 5 root root 1180 2022-06-08 15:17 ..
.....
lrwxrwxrwx 1 root root   21 1970-01-01 08:00 system_a -> /dev/block/mmcblk0p34
lrwxrwxrwx 1 root root   21 1970-01-01 08:00 system_b -> /dev/block/mmcblk0p35
lrwxrwxrwx 1 root root   21 1970-01-01 08:00 vendor_a -> /dev/block/mmcblk0p36
lrwxrwxrwx 1 root root   21 1970-01-01 08:00 vendor_b -> /dev/block/mmcblk0p37
```

***

# 2. Android badblock磁盘坏道检测工具

1.Android提供了badblock工具，检查emmc磁盘是否有坏道，可以以读的方式检查，也可以以写的方式检查

代码路径:`./external/e2fsprogs/misc/badblocks.c`

2.默认不编译，可在编译mk中配置：`PRODUCT_PACKAGES += badblocks`

3.然后单独编译，在system/bin下获取到该结果文件：

```shell
source ./build/envsetup.sh
lunch platform_name        ###(platform_name对应的sdk）
mmm external/e2fsprogs/misc/
```

4.把out/target/product/platform_name/system/bin/badblocks通过adb push到设备/system/bin/目录下

## 2.1. 命令检测方法

```shell
# adb shell
# badblocks -h
badblocks：选项需要一个参数 -- h
Usage: badblocks [-b block_size] [-i input_file] [-o output_file] [-svwnf]
       [-c blocks_at_once] [-d delay_factor_between_reads] [-e max_bad_blocks]
       [-p num_passes] [-t test_pattern [-t test_pattern [...]]]
       device [last_block [first_block]]
```

**常用参数含义：**

```shell
-b block-size: 以字节为单位, 指定区块的大小, 注意这是指每次的读(写)大小, 修改并不影响总的读(写)量
-i input_file: 读入一个已知的坏块列表。 Badblocks 命令将会跳过对这些已知是坏块的区块检查。如果 input_file 参数是“-”，则列表从标准输入读入。 在这个列表中列举出的区块也会在 新的 坏道记录文件或者坏道记录输出时被忽略掉。 dumpe2fs(8) 的 -b 选项能够在一个已有的文件系统中得到被标记为坏块的列表，而且已经做成了符合这个选项的格式。
-o output_file: 将坏块的列表写到指定的文件中。如果没有这个选项， badblocks 命令会在标准输出中输出这个列表。
-n: 使用非破坏性的读写模式。默认值是非破坏性的只读模式测试。这个选项不能与 -w 选项一起使用，因为它们是互斥的。
-s: 通过输出正在被检测的区块的号码以表示检测进程。
-v:  混杂模式检测。
-w:  使用写模式测试. 这个参数会破坏硬盘上的原有数据. 通过使用这个选项 badblocks 通过往每个区块上写入一些特定的字符（0xaa，0x55，0xff，0x00），读出来后再比较其内容，决定是否为坏块。 这个选项不能与 -n 选项一起使用，因为它们是互斥的。
-f: 正常情况下，badblocks命令不会在一个已经激活的设备上读写模式或者是非破坏性的读写模式的检测，因为这可能会导致系统的崩溃。 使用 -f 标志可以使这种情况强制执行，但是最好不要在正常的情况下使用它。如果/etc/mtab文件发生了错误，而设备实际上并没有被激活的时候，这个 参数才会是安全的。
-c number of blocks: 每一次检测区块的数目。默认值是16。增加这个数目可以增加检测 坏块 的效率可同时也会增加内存的耗费。 Badblocks 命令在只读模式下需要花费与每一次检测的区块相同数目的内存容量。在读写模式下，这个比例是两倍而在非破坏性的读写模式下，这个比例是三倍。

device [last_block [first_block]]
[磁盘装置][磁盘区块数 [启始区块]]
```

**典型命令：**

+ `badblocks -s -v -o sdbbadblocks.log /dev/sdb`: 对硬盘进行只读扫描，自动获取硬盘块数目并扫描全部块，将扫描日志输出到屏幕同时记录在sdbbadblocks.log文件中
+ `badblocks -w -s /dev/sdb END START`: 如果找到了坏道，可以进行写入扫描进行修复。写入扫描遇到坏道的时候会自动重映射。写入扫描会覆盖原有数据，所以请先备份。写入扫描速度很低，所以应该只扫描只读扫描时候发现错误的部分
+ 数据安全: `badblocks -n -b 4096 -c 16 -s /dev/sdx -o blocks-list`
+ 不保留数据: `badblocks -w -b 4096 -c 16 -s /dev/sdx -o blocks-list`
+ 指定数据: `badblocks -w -b 4096 -c 16 -s /dev/sdx -o blocks-list 122096645 15110746`
+ 读检测: `# badblocks -v /dev/sr0` （默认是只读检测）

***

# 3. linux e2fsck磁盘维护命令

e2fsck命令用于检查 Linux ext2 第二扩展文件系统的完整性，通过适当的选项可以尝试修复出现的错误

从实际调试过程中看，在Android项目中没有badblock方便，当然也可以尝试使用该命令进行检测

***

# 4. 问题调试检测方法

针对上面的问题现象，使用badblock进行检测：

1.只读扫描检测问题分区，检测到135个坏块

同步检测了system分区，也存在坏块（这两个分区数据量相比较大）

```shell
127|console:/dev/block/by-name # badblocks -sv vendor_a                                                         <
Checking blocks 0 to 524287
Checking for bad blocks (read-only test):
....
473282% done, 1:59 elapsed. (134/0/0 errors)
done
Pass completed, 135 bad blocks found. (135/0/0 errors)
```

2.使用写入扫描恢复：

```shell
console:/dev/block/by-name # badblocks -wv vendor_a
Checking for bad blocks in read-write mode
From block 0 to 2097151
Testing with pattern 0xaa: done                                                 
Reading and comparing: done                                                 
Testing with pattern 0x55: done                                                 
Reading and comparing: done                                                 
Testing with pattern 0xff: done                                                 
Reading and comparing: done                                                 
Testing with pattern 0x00: done                                                 
Reading and comparing: done                                                 
Pass completed, 0 bad blocks found. (0/0/0 errors)
```

3.再次只读扫描，未发现坏块。

4.尝试再次升级，此时注意需要使用和之前不一样的包升级（执行完整的升级过程，因为写入操作已经修改了分区数据）

5.仍旧出现分区IO读写错误

6.换不同的设备进行相同软件环境升级验证，验证该问题是否是个别机器复现，问题机器emmc硬件损坏。

从上面的调试结果看，软件层升级逻辑是正常的，怀疑emmc硬件损坏的可能性最大。

***

# 5. 参考

+ [调试笔记 --- eMMC坏块测试](https://blog.csdn.net/kris_fei/article/details/77316961?utm_source=app&app_version=5.3.1)
+ [Android性能分析之emmc坏块测试](https://blog.csdn.net/Ian22l/article/details/117085760?utm_source=app&app_version=5.3.1)
+ [badblocks坏道检测](http://t.zoukankan.com/52why-p-12363039.html)
+ [用badblocks检测硬盘坏道](https://www.mobibrw.com/2015/2003)
+ [Linux 磁盘维护 : e2fsck 命令详解](https://blog.csdn.net/yexiangCSDN/article/details/83181885)