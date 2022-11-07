---
layout: single
related: false
title:  Android RC文件分析
date:   2020-03-20 23:32:00 +0800
categories: android
tags: android
toc: true
---

> Android中最熟知的RC文件就是`init.rc`了，而在Hal接口服务定义中也会创建`.rc`文件。`init.rc`的语法分为行为(Actions),、命令(Commands) 、服务(Services)、选项(Options)。`.rc`文件是资源文件，包括比如对话框、菜单、图标、字符串等资源信息。使用`.rc`资源文件的目的是为了对程序中用到的大量的资源进行统一的管理。

<!--more-->

# 1. android rc文件分析

## 1.1. 模板

+ 结构：service关键字声明了你要定义一个service，而test就是这个service的名字，至于后面的目录则是这个service对应的可执行文件在系统中的位置（`adb shell`，即编译后的系统目录）。

+ init是分段(section)解析init.rc的，在`keywords.h`中可以查看关键字的定义。结合init.rc的内容，可以看出，init是以on 和 service来分段标记的。

```r
service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...
```

例如：
```r
service test /system/bin/vold \
        --blkid_context=u:r:blkid:s0 --blkid_untrusted_context=u:r:blkid_untrusted:s0 \
        --fsck_context=u:r:fsck:s0 --fsck_untrusted_context=u:r:fsck_untrusted:s0
    class core
    socket vold stream 0660 root mount
    socket cryptd stream 0660 root mount
    ioprio be 2
    writepid /dev/cpuset/foreground/tasks
```

**关键字解释：**

语法|关键字|说明
:-:|:-:|:-:
`SECTION`	|on|	触发条件
同上..	|service|	解析service
`COMMAND`|	chdir|	更改当前工作目录
同上..	|chroot|	更改参考的根目录位置
..  |class_start <serviceclass>|开启class start all services(启动某个设置了class名称的服务)
..	|class_stop <servicelass>|	停止某个设置了class名称的服务
..	|domainname <name>	|域名
..	|exec [ <seclabel> [ <user> [ <group> ] * ] ]  -- <command> [ <argument> ] *|	调用程序并转移进程(Fork一个进程然后执行命令)
..	|export|	提交变量
..	|hostname|	主机名
..	|ifup|	激活网卡
..	|insmod|	挂载模块(安装一个module)
..	|import <filename>|	引入init文件，比如etc下的一些rc文件，和java中的import差不多
..	|mkdir <path> [mode] [owner] [group]|	建立目录
..	|mount|	挂载文件系统
..	|setkey|	从源码看，应该是设置一个命令的关键字缩写，比如可以将domainname映射为dn
..	|setprop|	设置一个属性
..	|setrlimit	|设置当前程序可以打开的最大文件数到系统规定程序可以打开的最大文件数
..	|start|	启动服务
..	|stop|	停止服务
..	|symlink|	建立软链接
..	|sysclktz|设置基准时间
..	|loglevel|	Log输出级别，低于这个级别的就输出
..	|restart <service>|	重启服务,类似stop 但是不会disable service
..  |bootchart_init|开启bootcharting
..  |chmod <octal-mode> <path>|改变文件执行权限
..  |chown <owner> <group> <path>|改变文件的owner group
..  |enable <servicename>|将一个disabled的service变成enabled。且start
..  |load_all_props|加载system vendor的属性
..  |load_persist_props|加载data下面的persist属性
..  |mount_all <fstab>|挂载fstab中的设备
..  |`mount <type> <device> <dir> [ <flag> ]* [<option>]`|挂载设备
..  |powerctl|对sys.powerctl属性的respond
..  |`restorecon <path> [ <path> ] *`|恢复文件到sercurity context在file_contexts配置的
..  |`restorecon_recursive <path> [ <path> ]*`|递归的恢复目录中的文件到sercurity context
..  |`trigger <event>`|触发触发器
..  |`wait <path> [ <timeout> ]`|poll for 给定的文件 或者 timeout时间到。如果时间没有设定，默认为5秒
..  |`write <path> <content>`|打开文件，write string到给定文件。没有文件会被创建。有的话，会truncated
`OPTION`(用来初始化Service的)|	capability|	能力，也就是系统对进程的一种权限控制。
同上..	|class	|设置class name
..	|console|	启用控制台
..	|critical|	是否关键，也就是4分钟之内重启超过4次的话，重启之后就进入recovery模式
..	|disabled|	当它的class启动时，Service不会自动开启。必须显示的started by name(用其名字)
..	|group <groupname> [ <groupname> ]*	|组归属（改变username当执行这个Service之前）
..	|oneshot|	只启动一次，意外退出后不必重启
..	|onrestart|	执行一个命令，当Service重启时
..	|setenv	|增加环境变量
..	|socket|	申请socket资源
..	|user<username>|	用户归属（改变username当执行这个Service之前）
..	|ioprio	|io调度优先级
..  |writepid <file...>|当fork一个子进程时，写子进程的pid到一个给定的文件。是给cgroup/cpuset使用
..  |Triggers|Triggers被用来匹配事件，然后加入执行队列。
..  |boot|当init开启时，这是第一个执行的trigger


## 1.2. `class <name>`

> `class <name>`意思是为该服务定义一个类名，所有在这个类名下的服务都将一起启动和停止、
如果没有定义class选项，则默认`class deafult`。

定义为核心service，当`class core`服务启动时，这个vold启动。
如果是定义`class hal`，是不会自动启动的。可以定义为`class main`能够自动启动。

通过`adb shell ps -A|grep 关键字`查看进程服务。

## 1.3. `on <name>`

```r
on <trigger>
   <command>
   <command>
   <command>
```

on属于行为。

+ `on early-init`: init之前、加载完所有rc文件后即执行，init.rc在early-init执行的是`start ueventd`，根据keywords.h的定义，start是个命令(COMMAND)。
+ `on init`: 加载propety各项属性文件之前执行，在init变为propety service之前都属于init阶段。
+ `on early-boot`: 启动属性服务后即执行。
+ `on boot`: boot的时候执行。
+ `on property:xxxxx=x`: 当某个属性设置为预期值时执行。

