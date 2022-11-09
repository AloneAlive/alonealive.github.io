---
layout: single
related: false
title: Ubuntu18系统安装（无需制作启动盘）
date:   2019-07-25 21:31:37
categories: linux
tags: linux
toc: true
---

> 个人笔记本安装ubuntu实践，记录安装方式，无需制作启动盘

# 1. 数据备份

## 1.1. 准备工具

1. Ubuntu系统（官网选择版本下载ISO文件）
2. U盘（之前有一个U盘是做了window10的启动盘）

# 2. 安装步骤

1.移除备份好的机械硬盘

2.制作一个启动盘(win10系统不需要用软碟通UltraISO作启动盘)

```shell
（1）U盘最好是16g，USB3.0

（2）清空U盘（格式化）

（3）将ISO文件双击然后选中所有的文件将其复制到U盘（win10好像是不需要启动装置软件）

（4）U盘保持插到电脑上

（5）win10关闭快速启动选项，“电源选项”-》“选择电源按钮的功能”-》点击“更改当前不可用的设置”-》取消“启动快速启动”选项
```

3.修改启动项（保持U盘插到电脑上！）

```shell
（1） 重启电脑，一直按F2（或者F1），进入BIOS

（2） 点击方向箭头移动到Security，再向下移动到Secure Boot点击回车键，选择Disable

（3） 点击方向箭头移动到boot,查看USB Boot是否是Enalbed（有的电脑没有USB Boot，在Devices模块的USB Setup选项修改）

（4） 按F10保存退出
```

4.安装

```shell
（1）上一个步骤保存后还是会进入windows，再重启一遍（U盘不要拔出来，如果光驱位有机械硬盘记得拔出来）

（2）重启一直按F12，进入Boot Option Menu开机选择界面

（3）选择U盘（USB3.0/2.0这种标识）

（4）等待一会进入启动选项，先选择"try ubuntu without install",查看ubuntu是否可以安装在电脑上。

（5）然后会进入Ubuntu桌面，如果觉得正常，可以安装，就点击左上角的"install ubuntu 18.04LTS"（版本选择最新的稳定版本）
```

5.安装Ubuntu预配置

```shell
（1）选择语言（建议中文或者英文，进入系统后可以切换语言）

（2）键盘布局（建议选择汉语）

（3）无线选项，暂时不连接wifi

（4）更新和其他软件选项，选择“最小安装”（避免系统自动安装多余软件，加快速度）

（5）安装类型选项，此处会选择是否删除之前的系统和硬盘所有的文件！
    --- 如果避免出现问题，可以选择“其他选项”；
    --- 如果需要全新的安装，就选择删除之前的系统的选项；（我选择的这一项，然后系统会默认分区，就不需要之后两步）

（6）开始分区，重新分区或者分一块空闲盘的内存（建议不少于20G）
    --- EFI分区，存放系统引导文件的引导分区（可以默认选择，或者512MB）
    --- 交换分区swap，逻辑分区，建议和电脑内存大小相同（例如我的是8G）
    --- 根分区“/”，存放apt-get安装的软件，这个大小自己分配（建议内存大一点，以便安装软件有足够的空间）
    --- "/home"分区，存放个人文件，剩余的磁盘空间可以都给它
    （这样分区的话，如果重装Linux系统只需要格式化根分区，而保留“/home”分区）

（7）接着在安装类型选项界面下侧，有一个“安装启动引导器的设备”，选择刚刚分配好的EFI分区

（8）选择“安装”
```

6.安装进行时

```shell
（1）选择时区
（2）设置用户名、密码
```

7.安装后善后

```shell
（1）如果安装后无法关机，直接长按电源键强制关机，然后再次重启，查看是否正常；

（2）开机一直按F12进入开机选择界面选择Ubuntu/或者一直按F2进入启动项修改第一启动项为Ubuntu

（3）如果内存大于等于8G，交换分区swap分不分配也没有问题

（4）显卡驱动不生效：开机按F2进入启动项，进入BIOS中关闭，再重启，显卡驱动就会生效了（这个应该是内核或者驱动和电脑的安全模式启动冲突了）

（5）如果安装的时候一直卡在语言包下载，则点击右侧的skip跳过

（6）如果vi编辑器输入错乱，编辑/etc/vim/vimrc.tiny,将'compatiable'修改成

'nocompatiable',然后加入一行'set backspace=2',保存退出（可以sudo使用vim编辑，或者root权限使用gedit编辑）
```

***

# 3. Ubuntu桌面主题修改

## 3.1. Gnome桌面环境查看

```shell
在登录界面的右上角查看小图标，选择GNOME图标

或者安装：
** sudo apt-get install gnome-shell  （窗口管理程序）
** sudo apt-get install gnome-tweak-tool
```

## 3.2. 修改主题(只在当前的登录用户)

```shell
（1）打开文件目录tweek tool，查看到Extensions
（2）查看插件选项是否有User themes插件
    --- 如果没有tweek tool, 执行sudo apt-get install gnome-tweek-tool
    --- 如果没有use themes, 执行sudo apt-get gnome-shell-extensions

（3）登录GNOME主题官网(https://www.gnome-look.org)，选择主题下载
（4）下载后解压放到“~/.themes”目录，如果没有，自己创建
（5）重新启动tweak tool（可能是重启窗口），然后在Appearance中选择“Shell theme”，修改即可
（6）按”alt+F12“弹出命令窗口，输入“r”，重启Gnome桌面环境，查看修改后的主题
```

## 3.3. 图标和指针的主题

```shell
修改方式和上面一样，但是下载解压后的文件夹放置位置不同。
图标： /usr/share/icons/   （全局修改）
指针： ~/.icons/        （只限当前用户，即/home/account/）
```

***

# 4. ibus输入法有些软件不能输入中文

```shell
ibus-daemon -drx

此时终端不能编辑，重启终端

之后就可以输入中文。
```

# 5. 终端主题zsh修改

```shell
1. 查看当前shell主题

echo $SHELL

2. 查看shell列表
cat /etc/shell

3. 安装zsh
再次查看列表会有zsh的存在

4. 切换shell
chsh -s /bin/shells

5. 重启服务器
```

# 6. 添加开机启动程序的自定义命令

```bash
vi /etc/rc.local

例如ibus输入中文： ibus-daemon

ibus-daemon -dxr 重启ibus
```

# 7. 查看CPU使用情况

> cat /proc/cpuinfo

# 8. 查看磁盘内存使用情况

```shell
df -h

df -i 检查inode使用情况

sudo du / -h --max-depth=1
```
