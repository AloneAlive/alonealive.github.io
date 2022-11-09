---
layout: single
related: false
title:  Android Framework 开发调试技巧（十月份更新）
date:   2019-10-12 23:52:00
categories: android
tags: debug
toc: true
---

> Android系统framework开发（包含kernel、graphics、adb、性能、display等）调试技巧笔记


# 1. readelf命令查看ELF格式的文件信息

常见的文件比如动态库(`*.so`)、静态库（`*.a`），常用命令：`readelf -a libgui.so | grep test_string`

# 2. 查看手机内存

```shell
adb shell
cat proc/meminfo

MemTotal:        7821184 kB
MemFree:          157484 kB
MemAvailable:    2765976 kB
Buffers:          175624 kB
Cached:          2264796 kB
......
```

# 3. adb logcat缓存管理

```
adb root;adb remount

清除缓存  adb logcat -c

如果不能清除，就指定区域清除  adb logcat -c main/system/event/kernel/all(日志缓冲区)

查看缓存  adb logcat -g
设置最大logcat缓存   adb logcat -G 100M
```

# 4. linux离线翻译工具`sdcv`

```
安装：
sudo apt-get install sdcv

使用：
sdcv

只能翻译单词
```

# 5. apt-get命令列出版本号

`sudo apt-cache madison  openssh-client`

# 6. Linux安装tar.xz软件包

```
在linux下(node.js)：
解压后将bin目录的命令全局，建立软链接：
sudo ln -s /node-v10.16.0-linux-x64/bin/node /usr/local/bin/node
sudo ln -s /node-v10.16.0-linux-x64/bin/npm /usr/local/bin/npm
```

# 7. JNI的jlong类型打印

不使用%lld和%ld，而是先将其转换成long，然后%ld打印。

```c++
ALOGD("Mylog: JNI destroySurface nativeObject=%ld", (long)nativeObject);
```

***

# 8. monkey测试

## 8.1. 指令参数

```shell
adb shell monkey help查看帮助

-p 指定包名
-v log详细程度（最高支持-v -v -v）
-s 种子（指定后，同一个命令在任意时间地点的执行顺序相同）
--throttle 单步延时（每步操作间隔，单位ms），如果不知定系统会尽快的发送事件序列
--kill-process-offer-error 出错是杀掉进程
--ignore-timeouts 忽略超时错误(程序未响应)，不会应ANR而停止
--ignore-security-exceptions 忽略证书或认证异常 
--hprof 测试前后会生成app内存快照文件（一般在/data/misc目录下生成hprof文件，可以使用Android Studio查看）
--ignore-crashes 忽略crash，不会因crash而停止
--ignore-native-crashes 忽略native代码发生的crash崩溃，不会因此停止
--monitor-native-crashes 监视native代码发生的崩溃
```

## 8.2. 示例

(1) adb shell monkey -p PackageName -v -v -v -s 12 --throttle 500 1000 > monkey.txt
随机数种子是12，log详细程度最高，单步延时500ms，总执行1000步，日志输出到monkey.txt

(2) adb shell monkey -p PackageName --throttle 200 --ignore-security-exceptions -v 100000000

(3) adb shell monkey -p PackageName --throttle 200 --ignore-crashes --ignore-timeouts --ignore-security-exceptions --ignore-native-crashes --monitor-native-crashes -v -v -v 1000000 > monkeylog.txt

(4) 不指定包：
adb -s DeviceID shell monkey --throttle 200 --ignore-crashes --ignore-timeouts --ignore-security-exceptions --ignore-native-crashes --monitor-native-crashes --ignore-security-exceptions -v -v -v 100 > monkeylog.txt

(5) 正常测试验证问题使用（不忽略crash,压力测试，所以不指定间隔时间）：
adb -s DeviceID shell monkey -p PackageName --throttle 200 --ignore-security-exceptions -v 100000000 > monkeyLog.txt

## 8.3. 停止monkey测试

```shell
(1) ps -A|grep monkey
(2) kill pid（11111）
```

## 8.4. 重现monkey测出的bug

monkey日志搜索`ANR exception`，将之前的事件重新操作。尤其是seed值要一样，如`monkey -p PackageName -v seed(-s) 100(seed的值) 500(随机时间次数)`

***

# 9. dump meminfo

```shell
adb shell dumpsys meminfo
或者具体包
adb shell dumpsys meminfo packageName
```

***

# 10. kernel使用printk调试

+ 打印调试log

`printk("%d",intA);`

+ 打印变量所占内存大小

`printk("sizeof(*intA)=%d",sizeof(*intA);`

# 11. 查看设别是否支持Project Teable

Project Treble是在最新的Android上应用兼容的芯片驱动，加快最新系统适配的速度。

`adb shell getprop ro.treble.enabled`

# 12. 查看cpu架构

`adb shell getprop ro.product.cpu.abi`

# 13. Service服务命令

```
服务列表：    adb shell service list
检测某服务是否存在：   adb shell service check SurfaceFlinger
```

# 14. 测试CtsMediaTestCases需要CTS媒体文件（连接外网）

Google官网下载，例如CTS媒体文件1.4，解压后阅读README文件，按照提示copy文件到device。

```
chmod 544 copy_media.sh
./copy_media.sh
chmod 544 copy_images.sh
./copy_images.sh

例如：run cts-dev --module CtsMediaTestCases --compatibility:module-arg CtsMediaTestCases:include-annotation:android.platform.test.annotations.RequiresDevice
```

***

# 15. 当前活动Activiy

## 15.1. 获取当前ACTIVITY

`adb shell dumpsys activity top|grep ACTIVITY`

## 15.2. 命令启动指定Activity

`adb shell am start -n ActivityName`

***

# 16. Android SDK抓取systrace

```shell
进入Android/Sdk/platform-tools/systrace目录下
python  systrace.py -b 8000 -t 5 -o systrace.html
```

## 16.1. 打开模拟Vsync，从Systrace查看到

+ 源码： Android 10的AOSP
+ 方法： 修改`surfaceflinger/Scheduler/DispSync.cpp`的`static const bool kEnableZeroPhaseTracer = false;`为True

# 17. adb devices很少识别

```shell
方法一：
lsusb  查看usb是否识别到手机
如果没有，检查“开发者选项 -> USB调试”是否打开

方法二： 重启ADB服务
adb kill-server
adb start-server
```

## 17.1. 添加usb设备自动识别信息/etc/udev/rules.d/

```shell
（1）lsusb查看usb链接设备
（2）然后编辑/新增文件(/etc/udev/rules.d/)：
sudo vi 51-android.rules 文件内容：
SUBSYSTEM=="usb",ATTRS{idVendor}=="093a",ATTRS{idProduct}=="120d",MODE="0666"
（3）保存后重启udev服务
sudo /etc/init.d/udev restart或者sudo service udev restart
（4）重新连接设备测试
```

# 18. adb shell获取设备信息参数（序列号）

```shell
getprop 查看机器的全部信息参数
getprop ro.serialno 查看机器的序列号
getprop ro.carrier 查看机器的CID号
getprop ro.hardware 查看机器板子代号
getprop ro.bootloader 查看SPL(Hboot)版本号
getprop ro.build.version.release 查看系统版本（8、9...）
getprop ro.build.display.id  获得厂商系统版本
```

# 19. Android中CPU频率查看和修改

```shell
root权限（直接输入su命令）
cd sys/devices/system/cpu/cpu0/cpufreq
ls文件如下

cpuinfo_cur_freq： 当前cpu正在运行的工作频率
cpuinfo_max_freq：该文件指定了处理器能够运行的最高工作频率 （单位: 千赫兹）
cpuinfo_min_freq ：该文件指定了处理器能够运行的最低工作频率 （单位: 千赫兹）
```

# 20. Ubuntu下载更新杀毒软件

```shell
下载：
sudo apt-get update
sudo apt-get install clamav

sudo chmod 777 freshclam.lg
freshclamg更新

扫描：
clamav -a /
```

# 21. 安装deb包

> sudo dpkg -i gapid-1.3.1-linux.deb

# 22. 查看手机服务

> 开发者选项 -> Running services可以查看正在运行的服务，以及运行内存情况

# 23. 查看修改屏幕分辨率和密度

```shell
查看：
adb shell
wm   size  获得手机当前分辨率
wm  density   获得手机当前屏幕密度（例如560dpi）

修改：
wm size 1096*2560
wm density 420

恢复：
wm size reset
wm density reset
```

# 24. 查看进程map虚拟地址

```shell
adb root;adb remount
adb shell  
ps -A|grep camera 查看服务的进程号(例如4712)  
cd proc/4712  进入进程号的文件夹  
more maps 查看虚拟内存地址  
```

# 25. SELinux模式开启关闭

SELinux(Security-Enabled Linux)是美国国家安全局（NSA）对于强制访问控制的实现（安全子系统）

临时生效方法：

+ `adb shell setenforce 0（临时生效，关闭SELinux模式）`
+ `adb shell setenforce 1（启用，开启SELinux模式）`

***

# 26. adb命令 -- 录制手机视频

+ `adb shell screenrecord sdcard/record.mp4`  
+ `1adb pull sdcard/record.mp4 .`

# 27. adb命令 -- 截图

+ `adb shell screencap -p sdcard/1.png`

# 28. adb命令 -- 输入文本

节省手动输入的时间：
+ `adb shell input text ****`

# 29. adb命令 -- 获取APP路径

+ `adb shell dumpsys SurfaceFlinger  最下方查看正在运行的APK`
+ `adb shell pm path "com.**" 获取路径`

# 30. 台式机通过WIFI建立adb连接，实现无线连接手机

通常在需要手机连接外设显示设备的同时需要抓取Log、Dump等操作：
1. 两台手机连上同一个无线网
2. 其中一台A关闭开发者选项的USB调试，并且连接到电脑作为热点（无线网卡）
3. 另一台B连接电脑输入`adb tcpip 5555`，然后输入`adb shell ifconfig wlan0`查看B的IP地址
4. 连接B手机的IP： adb connect IP
5. 断开B，保持A连接在电脑

# 31. 连接DP继续dump SurfaceFlinger的方法

1. 连接DP前启动：
`adb shell dumpsys SurfaceFlinger --file --no-limit`
2. 断开DP后再次执行结束Dump：
`adb shell dumpsys SurfaceFlinger --file --no-limit`
3. 接过文件pull出来：
`adb pull /data/misc/wmtrace/dumpsys.txt`

***

# 32. md5sum命令检测文件

通过`md5sum filename`查看文件的`md5sum值`是否一样

***

# 33. GSI含义

> GSI是替换成google的frameworks等(即system.img, 即google的原生AOSP)
> system.img包含整个系统，framework、application等，被挂接到”/“目录下，包含系统的所有二进制文件。大概是编译出来的out/target/product/ProductName/system目录的映射

## 33.1. GSI方法

```shell
常用步骤：
adb reboot bootloader   (fastboot devices检测设备)
fastboot erase system
fastboot -w
fastboot flash system GSI(.img file)
fastboot reboot  
```

***

# 34. CTS VTS跑测注意点

+ 重跑：`run retry --retry 序列号`
+ 跑测arm64-v8a还是armeabi-v7a等:`run cts-suite -s ... -a arm64-v8 -m ...`

# 35. linxu命令查找命令

## 35.1. 查找字符串

+ `grep -rn  字符串`
+ `grep 字符串 -Rin *    //查找该目录下包含该字符串的文件`

## 35.2. 查找文件

+ `find . -iname Test*`

***

# 36. Git命令

1. 回退到某个commitID： `git reset --hard commitID`
2. 新建一个Commit用于Revert某个分支： `git revert commitID`
3. 修改commit信息
`git commit --amend --author="**" --date="**" 修改作者和日期`
4. 添加topic的方法(提交到Gerrit)
`git push origin HEAD:refs/for/BRANCH_Name%topic="name"`
5. 使用新的change ID覆盖原来已经提交的Patch
`git push origin HEAD:refs/changes/99999 (gerrit上的已有Commit的Patch ID)`


# 37. **`git stash`暂时储藏**

用于修改的代码暂时保存起来，并且不影响下次的修改(这个比生成本地补丁方便`git format-patch -1 commitID`)：
+ `git stash save "Remarks"` 执行存储，并且添加备注
+ `git stash list` 查看储藏列表
+ `git stash show stash@{1}` 查看某次储藏的修改
+ `git stash apply stash@{$Num}`  应用某次储藏
+ `git stash pop stash@{$Num}`  从缓存堆栈中删除某次储藏并且应用到代码中，默认第一个stash
+ `git stash drop stash@{$Num}` 从列表删除这个存储
+ `git stash clear`  删除所有缓存的stash

***

# 38. Repo下载项目方法

1. 下载`.repo`：
`repo init -u Git远程仓库 -b Branch_Name(分支选择)`
2. Sync项目：`repo sync -c -j4`（当前分支）
3. 如果值Sync指定目录，则在指定目录下执行：`repo sync -c . -j4`

# 39. Android常用编译方式

1. `source build/envsetup.sh`
2. `lunch //选择指定Product`
3. `make fullbuild -j4`
4. 如果只编译部分模块：

+ framework模块：`make framework -j4`，结果生成framework.jar包到/system/framework
+ 编译`kernal/msm`使用： `make bootimage -j4`
+ 指定目录下执行:`mm/mmm/mma(依赖编译)`

***
