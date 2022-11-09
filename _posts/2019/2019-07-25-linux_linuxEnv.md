---
layout: single
related: false
title: Ubuntu安装后的环境配置注意点
date:   2019-07-26 21:31:37
categories: linux
tags: linux
toc: true
---

> Ubuntu系统刚安装完成后，工作还只是进行了一小半，还有一大堆的环境需要配置搭建。比如说JDK, nodejs, python, vs code编译器 ...

<!--more-->

# 1. apt查询包的版本

> `apt-cache madison <<package name>>`

# 2. git配置

1.安装： `sudo apt-get install git`

2.config配置：  

`git config --global user.name "username"`  

`git config --global user.email "username@example.com"`  

3.ssh公钥生成：  

`ssh-keygen` 或者 `ssh-keygen -t rsa -C ****@**.com`

4.然后会生成.ssh/id_rsa.pub文件 将其拷贝到需要的地方即可（github）

# 3. vscode下载

```shell
sudo add-apt-repository ppa:ubuntu-desktop/ubuntu-make
sudo apt-get update
sudo apt-get install ubuntu-make

umake ide visual-studio-code

reboot   //如果安装完成没有vscode，就重启电脑
```

# 4. gnome主题切换

__类似以下的方法：__

1.pop命令安装

```shell
sudo add-apt-repository ppa:system76/pop
sudo apt update
sudo apt install pop-theme

//重启
alt+f2输入r
或者reboot
```

2.在[gnome主题网站](https://www.pling.com/s/Gnome/browse/)下载安装

下载后的压缩包解压到用户目录`.local/themes或者.local/icons/`目录下，之后使用`gnome tweaks tool`修改即可

# 5. npm install一直卡住

输入： `npm config set registry http://registry.cnpmjs.org`

# 6. 截图工具

`sudo apt-get install shutter`

# 7. java环境

`sudo apt install default-jdk`

`sudo apt install default-jre`

___测试结果：__

`java --version`

`javac`