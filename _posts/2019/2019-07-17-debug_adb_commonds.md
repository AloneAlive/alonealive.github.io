---
layout: single
related: false
title: Use ADB
date:   2019-07-17 22:04:14
categories: android
tags: debug
toc: true
---

> ADB是连接手机设备和电脑设备的调试桥。这种工具命令用于Android调试是基础而且重要的。

# 1. Overview

1. Install: sudo apt-get install adb
2. Function:

```shell
通过adb可以管理、操作模拟器和设备，如安装软件、查看设备软硬件参数、系统升级、运行shell命令等。
手机启动USB调试模式，设备连接电脑。
注：我的手机一加6的USB调试模式打开方式如下：
       （1）在手机设置的关于手机找到版本号，双击七次打开开发者模式;
       （2）在开发者选项中打开USB调试选项;
       （3）设备连接
```

***

# 2. <font color=red>Commonds</font>

## 2.1. 基础命令

> 如果设备连接>1，可使用adb -s DevicesID + 其余部分命令

Commond | Explain
:-:|:-:
adb devices | 查看设备连接情况
adb version | 查看版本
adb help                 |     	查看帮助
adb shell		   	|	进入adb sehll命令
adb shell top　　	|		查看手机当前进程占用手机内存情况 
adb shell kill -3 pid 	|	杀掉进程
adb logcat -v process &#124; grep 8607  | 8607是进程 PID
adb shell reboot -p | 关机
adb reboot  | 重启
adb shutdown | 关机
adb root | root
adb remount | 获取读写权限
adb kill-server | 关闭adb服务
adb start-server	|	启动adb服务
adb shell stop | 关闭设备请求
adb shell start |启动设备请求
adb shell su root setenforce 0 | 关闭seLinux模式

## 2.2. 网络相关设置

Commond | Explain
:-:|:-:
adb shell ifconfig | 查看手机IP
adb tcpip 5555 | 设置手机tcpip
adb connect IP | 连接IP

## 2.3. 屏幕display信息

Commond | Explain
:-:|:-:
adb shell wm size | 查看分辨率
adb shell set wm size | 设置分辨率


## 2.4. 软件操作命令

1. 安装软件:

```shell
adb install  	
adb install <apk文件路径> :这个命令将指定的apk文件安装到设备上
adb install -r -d APK
```

2. 卸载软件

```shell
adb uninstall <软件名>
adb uninstall -k <软件名>     如果加 -k 参数,为卸载软件但是保留配置和缓存文件
```

## 2.5. 文件操作命令：

1. 从电脑上发送文件到设备,用push命令可以把本机电脑上的文件或者文件夹复制到设备(手机)

```shell
adb push <本地路径> <远程路径>
```

2. 从设备上下载文件到电脑,用pull命令可以把设备(手机)上的文件或者文件夹复制到本机电脑

```shell
adb pull <远程路径> <本地路径>
```

## 2.6. 录制视频

```shell
adb shell screenrecord sdcard/test.mp4
adb pull sdcard/test.mp4 .
```

## 2.7. __日志操作命令__

1. __常用日志命令__

Commond | Explain
:-:|:-:
adb logcat &#124; tee log1 | 保存日志到本地并且打印到控制台
adb logcat -v threadtime | 按照线程时间打印日志
adb logcat -s System.out  	|	设置标签（某个字符串），过滤显示日志
adb logcat -c |	清理已存在的日志
adb logcat -g |	打印日志缓冲区的大小
adb logcat  >  home/mylog.txt	| 日志保存到电脑某路径
adb logcat  -d  -f  /sdcard/mylog.txt	| 保存到手机上指定位置（-d  日志显示在控制台）
adb logcat -f /scard/log.txt |	输出到手机指定位置

2. 查看日志

```shell
adb logcat

日志等级（由上往下级别递增）：
V   	verbase，级别最低，琐碎、不重要的日志信息
D 	debug，调试信息
I	info，重要信息
W	warning，警告信息
E	error，错误信息
F  	fatal，严重错误信息
S	slient，无记载
```

3. 查看帮助信息，获取该命令可配置的参数选项

```shell
adb logcat  --help    	
```

4. 加载一个可使用的日志缓冲区供查看

```shell
adb logcat -b <buffer> 

buffer选项可以填为：
radio	通信系统
system	系统组件
event	event事件模块
main	java层
kernel	linux内核
all     所有
```

5. 指定标签，指定级别过滤显示日志

```shell
adb logcat [tag:level]
例如：adb logcat Test:I
```

6. 设置日志输入格式控制输出字段

```shell
adb logcat -v <format>

format选项可以填为：
brief	显示优先级/标记和原始进程的PID（默认）	
process	只显示进程PID
tag		只显示优先级/标记
thread	只显示进程、线程、优先级/标记
raw
time
long

例如：adb logcat -v process
```

## 2.8. APK包相关命令

__adb shell pm命令__

```shell
adb shell pm -l 列出包列表
adb shell pm list packages	查看包名(同上)

adb shell pm path "PackageName"  获取包的路径(可以通过dump SF获取当前的活动包名)
adb shell pm list  packages  -f	查看包名对应的apk路径及名称
adb shell dumpsys　　列出手机所有apk的详细信息
```

## 2.9. 按名称检查正在运行的进程

```shell
adb shell pidof ”mediaserver“      //查找正在运行的服务名的pid
adb shell pidof com.android.phone  //获取进程号(如果找到此类进程，则返回PID，否则返回空字符串)
```

## 2.10. 获取当前ACTIVITY

```shell
adb shell dumpsys activity top|grep ACTIVITY
```

## 2.11. 命令启动指定Activity

```shell
adb shell am start -n com.android.systemui/com.android.systemui.recents.RecentActivity
```

## 2.12. 模拟点击事件（点击屏幕）

> 打开开发者选项的指针位置选项，点击屏幕即可获得XY坐标。

```shell
adb shell input tap x y
```

## 2.13. 获取设备信息参数

```shell
adb shell getprop "name"
例如：
adb shell
getprop 查看机器的全部信息参数
getprop ro.serialno 查看机器的序列号
getprop ro.carrier 查看机器的CID号
getprop ro.hardware 查看机器板子代号
getprop ro.bootloader 查看SPL(Hboot)版本号

setprop <参数名> <参数值>    设置某个参数
```

## 2.14. 输入命令input

> input后可以跟很多参数， text相当于输入内容，keyevent相当于手机物理或是屏幕按键，tap相当于touch事件，swipe相当于滑动。

1. 模拟的是滑动事件

```shell
input swipe模拟的是滑动事件，需要将起始的坐标传进去,可以传入滑动时长
adb shell input swipe <x1> <y1> <x2> <y2> [duration(ms)] (Default: touchscreen)

例如：

向左滑动：
input swipe 600 800 300 800

向右滑动：
input swipe 300 800 600 800

滑动：
adb shell input swipe 100 100 200 200 300 //从 100 100 经历300毫秒滑动到 200 200 

长按：
adb shell input swipe 100 100 100 100 1000 //在 100 100 位置长按 1000毫秒
```

2. 输入文本内容

```shell
adb shell input "text"   输入文本内容
```

## 2.15. adb shell am instrument …

instrument为am命令的一个子命令。用于启动一个Instrumentation测试。
各项参数：

```shell
-r: 以原始形式输出测试结果；该选项通常是在性能测试时与[-e perf true]一起使用。

-e  name value: 提供了以键值对形式存在的过滤器和参数。
例如：-e testFile <filePath>（运行文件中指定的用例）；-e package <packageName>（运行这个包中的所有用例）……  有十几种。 

-p file:    将分析数据写入 file。

-w: 测试运行器需要使用此选项。
例如:-w <test_package_name>/<runner_class> ：<test_package_name>和<runner_class>在测试工程的AndroidManifest.xml中查找，作用是保持adb shell打开直至测试完成。

--no-window-animation:  运行时关闭窗口动画。

--user user_id | current:   指定仪器在哪个用户中运行；如果未指定，则在当前用户中运行。
```

例如运行一个类中的所有用例：

```shell
adb shell am instrument -w -r -e class com.letv.leview.setproxy com.le.tcauto.uitest.test/android.support.test.runner.AndroidJUnitRunner
```

***

# 3. <font color=red>Dumpsys信息</font>

## 3.1. 服务列表信息

1. adb shell   　　进入shell
2. dumpsys -l 　　查看所有正在运行的服务名
3. service list  　　查看这些服务名称调用了哪个服务

## 3.2. 常见服务信息列表

服务名 | 类名 | 功能
:-:|:-:|:-:
gfxinfo | GraphicsBinder | 图像
SurfaceFlinger || 图像相关
activity | ActivityManagerService | AMS相关信息
package | PackageManagerService |PMS相关信息
window | WindowManagerService |WMS相关信息
input | InputManagerService | IMS相关信息
power | PowerManagerService | PMS相关信息
batterystats | BatterystatsService | 电池统计信息
battery | BatteryService | 电池信息
alarm | AlarmManagerService | 闹钟信息
dropbox | DropboxManagerService | 调试相关
procstats | ProcessStatsService | 进程统计
cpuinfo | CpuBinder |CPU
meminfo | MemBinder | 内存
dbinfo | DbBinder | 数据库
appops || app使用情况
permission || 权限
processinfo || 进程服务
batteryproperties || 电池相关
audio || 查看声音信息
netstats || 查看网络统计信息
diskstats || 查看空间free状态
jobscheduler || 查看任务计划
wifi || wifi信息
diskstats || 磁盘情况
usagestats || 用户使用情况
devicestoragemonitor || 设备信息

## 3.3. dump方法

```shell
dumpsys <service>　　　　打印具体某一项服务（service就是前面表格中的服务名）

例如： （adb shell）
dumpsys cpuinfo //打印一段时间进程的CPU使用百分比排行榜
dumpsys meminfo -h  //查看dump内存的帮助信息
dumpsys package <packagename> //查看指定包的信息
dumpsys SurfaceFLinger    //查看SF服务
```

## 3.4. dump窗口信息

```shell
adb shell dumpsys window windows
adb shell dumpsys window windows |grep Current   //当前窗口信息
```

## 3.5. __dump SurfaceFlinger信息__

1. 方式：

```shell
adb shell dumpsys SurfaceFlinger

一般包含：
1、layer的信息，layer一般对应于一个surface;
2、opengl的信息。一般是跟gpu比较相关的参数，opengl是标准的接口;
3、display。安卓支持三种类型的display，可以导出display当前的显示状态，也就是各个surface(layer)在各个display的显示属性;
4、surfaceflinger管理graphis buffer的信息。主要是layer申请的帧数据内存;
5、hwcomopser的如果实现dump接口也能知道hwcomposer的一些参数;
6、gralloc的内存分配信息。如果gralloc有实现dump接口的话;
7、合成方式、BufferLayer的树形结构信息
```

2. 连接DP 然后继续dump  SurfaceFlinger信息

```shell
连接之前启动: adb shell dumpsys SurfaceFlinger --file --no-limit 
断开后再次执行 adb shell dumpsys SurfaceFlinger --file --no-limit 
并pull出来 adb pull /data/misc/wmtrace/dumpsys.txt 
```

## 3.6. GPU帧渲染数据

```shell
adb shell dumpsys gfxinfo 

例如查看camera功能的渲染一帧所经过的各个阶段的耗时情况（单位毫秒）
dumpsys gfxinfo **.camera 

带上 framestats 参数可以获取最近的 120 帧数据：
adb shell dumpsys gfxinfo **.camera framestats
```

# 4. fastboot模式

## 4.1. 进入fastboot（设备需要解锁）

> adb reboot bootloader
> fastboot devices 检查设备

## 4.2. 刷机

1. fastboot erase boot/system 清除区
2. fastboot -w
3. fastboot boot/system  NewImg  烧录
4. fastboot reboot