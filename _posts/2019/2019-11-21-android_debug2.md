---
layout: single
related: false
title:  Android Framework 开发调试技巧（十一月份更新）
date:   2019-11-23 23:52:00
categories: android
tags: debug
toc: true
---

> Android系统framework开发（包含kernel、git、adb、gdb、linux等）调试技巧笔记

# 1. 更新adb

+ 更新命令：   sudo apt-get install android-tools-adb
+ 查看当前adb指令的目录: which adb
+ 查看版本：adb version

***

# 2. adb shell相关

## 2.1. ps（正在运行的进程）

```shell
USER 	进程当前用户
PID 	进程ID
PPID 	父进程ID
VSZ	进程的虚拟内存大小，以KB为单位
RSS 	实际占用的内存大小，以KB为单位
WCHAN 	进程正在睡眠的内核函数名称；该函数的名称是从/root/system.map文件中获得的。
PC 			计算机中提供要从[存储器]中取出的下一个指令地址的[寄存器]
NAME 		进程状态及名称
```

## 2.2. top（CPU使用率）

> top命令提供了实时的对系统处理器的状态监视。它将显示系统中CPU最“敏感”的任务列表。该命令可以按CPU使用，内存使用和执行时间对任务进行排序。

1. `VIRT`：这个内存使用就是一个应用占有的地址空间，只是要应用程序要求的，就全算在这里，而不管它真的用了没有。写程序怕出错，又不在乎占用的时候，多开点内存也是很正常的;
2. `RES`：resident memory usage。常驻内存。这个值就是该应用程序真的使用的内存，但还有两个小问题，一是有些东西可能放在交换盘上了（SWAP），二是有些内存可能是共享的;
3. `SHR`：shared memory。共享内存。就是说这一块内存空间有可能也被其他应用程序使用着;
4. `DATA`：数据占用的内存。这一块是真正的该程序要求的数据空间，是真正在运行中要使用的。

## 2.3. vmstat（显示系统信息的）

vmstat是一个显示系统信息的命令。例如，它显示主存储器的可用容量和CPU的操作状态。
如果按原样执行vmstat命令，则会显示有关当前进程，内存，交换，设备，中断和CPU的信息。此外，如果附加“ - d”或“ - p”选项，将显示分区和磁盘上的读/写状态等。指定“-f”选项时，从系统启动到命令执行将显示创建进程的次数。

如果在vmstat之后指定以秒为单位的时间间隔，则每隔指定时间显示一次系统状态。此外，当您指定次数时，会显示指定的信息次数。

对于容量，可以使用“-S”选项指定单位。指定“-SM”时，容量单位以M字节显示。

例如：
以10秒为间隔显示内存和CPU信息三次： `vmstat 10 3`

## 2.4. meminfo（内存系统信息）

`cat /proc/meminfo`

## 2.5. free（显示内存使用情况）

> 可以知道当前的内存使用情况。

```shell
-b	以字节显示容量（默认）
-k	显示容量，以千字节为单位
-m	显示容量，以MB为单位
-h  显示容量单位，包含Ｇ、Ｍ
-t	还显示物理内存和交换内存的总和
```

***

## 2.6. strace（跟踪进程执行时的系统调用和所接收的信号）

> strace常用来跟踪进程执行时的系统调用和所接收的信号。 

在Linux世界，进程不能直接访问硬件设备，当进程需要访问硬件设备(比如读取磁盘文件，接收网络数据等等)时，必须由用户态模式切换至内核态模式，通 过系统调用访问硬件设备。strace可以跟踪到一个进程产生的系统调用,包括参数，返回值，执行消耗的时间。

通用的完整用法：
`strace -o output.txt -T -tt -e trace=all -p 12345`

上面的含义是跟踪28979进程的所有系统调用（-e trace=all），并统计系统调用的花费时间，以及开始时间（并以可视化的时分秒格式显示），最后将记录结果存在output.txt文件里面。

## 2.7. time（linux命令，ADB通用）

> 测量从调用指定命令到结束所花费的时间，用户CPU时间和系统CPU时间。在指定命令的输出结果之后，将测量结果输出到标准错误输出。命令代码实际使用CPU的时间是用户CPU时间。因此，如果将不存在的命令作为time命令的参数，则用户CPU时间变为0。睡眠时间不计算在内。

**例如：**
显示ls命令的执行时间： `time ls -a`

```shell
real	0m0.006s
user	0m0.006s
sys	0m0.000s
```

## 2.8. size

> 显示一个目标文件或者链接库文件中的目标文件的各个段的大小(可执行文件段的大小,默认为a.out)

例如（linux下）：   `size libui.so`

text表示正文段大小，data表示包含静态变量和已经初始化（可执行文件包含了初始化的值）的全局变量的数据段大小，bss由可执行文件中不含其初始化值的全局变量组成。

## 2.9. file（辨识文件类型）

> file确定并显示文件类型，例如可执行文件或文本或其他数据。

**例如：**

`file libui.so`
或者 `adb shell file ……`

```shell
-b	以简单模式显示
-i	使文件成为mime类型字符串
-z	还要检查压缩文件
-v	显示版本
```

***

# 3. fastboot相关

+ 重启进入Recovery界面： adb reboot recovery
+ 重启进入bootloader界面： adb reboot bootloader

## 3.1. 进入Recovery模式

1. 查看设备： adb devices
2. adb root
3. adb shell
4. 进入fastboot： adb reboot fastboot
5. fastboot devices 
6. 查看当前使用分区： fastboot getvar current-slot
7. 接着擦除分区和用户数据，然后flash烧录


如果不能识别或者没权限，优先检查lsusb添加序列号到`/etc/udev/rules.d/`

如果出现错误:`no permissions fastboot`

用fastboot命令查看设备提示无权限，如下：

```shell
fastboot -l devices
no permissions         fastboot usb:1.2-1
```

因为权限问题，是fastboot没有权限， 解决步骤：
1. 将fastboot的所有者属性改成root,用`which fastboot`命令找到fastboot所在的目录，然后进入此目录
2. 用命令chown改其属性:`sudo chown root:root fastboot`
3. 将其权限更改一下：`sudo chmod +s fastboot`

还存在一种可能性，就是`adb`版本过低。

***

## 3.2. 部分参数

fastboot [options]|Notes
:-:|:-:
-w |清空用户数据分区和缓存分区.相当于recovery中的"wipe data/factoryreset"
-s <串口号> |指定要操作的设备的串口号
-p <产品名> |指定要操作的设备的产品名.比如hero,bravo,dream...
-c <命令行> |用命令行替换系统的启动命令行

***

# 4. 解析so文件addr2line

`addr2line [address] -e test.so -f`  
或者  
`readelf -a [.so/.bin]`

## 4.1. 根据解析结果查询函数

C++在linux系统编译后会变成类似`_ZNK...`的修饰名。使用`c++filt`获取函数的原始名称：

`c++filt [_ZNK...函数修饰名]`

***

# 5. 跳过开机向导

`adb shell settings put global device_provisioned 1（默认是0）`

# 6. Android 10 AOSP源码打开模拟Vsync（Systrace可查看）

+ 源码： Android 10的AOSP
+ 方法： 修改`surfaceflinger/Scheduler/DispSync.cpp`的`static const bool kEnableZeroPhaseTracer = false;`为`True`
+ 另外在查看`ZeroPhaseTracer`还需要打开`mTraceDetailedInfo`，即`const bool mTraceDetailedInfo = true;`

# 7. 对比文件和文件夹区别（可用于git解决冲突）

`meld 文件/文件夹`

比较文件: `vimdiff a.txt b.txt`

# 8. repo

> Android代码包含几百个git库，下载和管理都需要一个方便的工具，Google开发了repo用来管理多个git库，通过manifest.xml文件将一个个的git库管理起来,形成一个系统。

## 8.1. Gerrit

> `Gerrit`是Google开发的一个代码审核工具。它是一个Web工具,它靠git来存放代码,靠repo这个接口来提交和下载修改。 提交到Gerrit时,每个Git库的修改都会变成一次提交,每个提交可以有一个或多个人来review和verify。当你的修改被批准之后,Gerrit会把修改真正提交到指定的分支中。

Gerrit上代码提交的三种状态：
`Open、Merged、Abandoned`

1. `Open`: 状态的代码需要经过Review,Verify,Submit操作后才会真正入库,即成为Merged状态
2. `Merged`: 状态的代码已经入库,不能再Abandoned,只能Revert
3. `Open`: 状态的代码由于各种原因不能入库的可以放弃,即Abandoned状态。Abandoned 状态的代码不能再入库,如有需要,可以“Restore”。

## 8.2. Jenkins

一个持续集成工具,一个运行任务的平台。能实施编译、监控集成中存在的错误，提供详细的日志文件和提醒功能。能用图表形象地展示项目构建的趋势和稳定性。

## 8.3. repo链接指定版本的`manifest.xml`

`repo init -m manifest_TEST.xml`
然后可以在目录查看结果，再同步代码。

***

# 9. ssh生成publickey（指定邮箱）

`ssh-keygen -t rsa -C ****@mail.com`

***

# 10. Git相关命令

## 10.1. git用户设置

+ `git config --global user.name ***`
+ `git config --global user.email ****@mail.com`

+ 查看config： `git config -l`

## 10.2. 生成补丁

`git format-patch -1 [最近的提交CommitID]`

## 10.3. 生成指定某个commit提交的补丁

`git format-patch abc123d^..abc123d`

## 10.4. 获取补丁

1. 申请生成在本地，但是没有加入暂存区： `git apply [PatchA]`
2. 直接申请生成提交： `git am [PatchB]`

## 10.5. git add用法

+ `git add . `:提交所有修改的文件,包括新增文件,不包括删除文件
+ `git add -u `:提交所有修改文件,包括删除文件,不包括新增文件
+ `git add -A `:提交包括新增和删除文件的所有文件

## 10.6. 从暂存区域移除等其他命令

Command|Notes
:-:|:-:
git rm|			            从暂存区域移除,并连带从工作目录中删除指定的文件
git rm -f|                   如果删除之前修改过并且已经放到暂存区域的话,则必须要用强制删除选项`-f`
git reset HEAD <file>… |	    取消对文件的修改,把之前版本的文件复制过来重写此文件。
git checkout -- <file>…	|	取消已经暂存的文件
git clean		        |	删除未暂存的文件
git diff	    		|	查看尚未暂存的文件更新了哪些部分(和暂存区中)
git diff –cached	    |	看已经暂存起来的文件和上次提交时的快照之间的差异
git log –graph			|	显示图形表示的分支合并历史
git log –since=2.weeks|		列出所有最近两周内的提交
git log –p	|				以patch形式显示提交
git log -p -2|常用 -p 选项展开显示每次提交的内容差异，用 -2 则仅显示最近的两次更新
git log --stat|仅显示简要的增改行数统计。

***

# 11. GDB命令

Command|Notes
:-:|:-:
bt| 查看各级函数调用及参数
bt full|详细堆栈信息
bt PID|查看PID信息
frame |选择栈帧
info locals| 查看当前栈帧局部变量的值
info registers |可以看函数入参
thread n| 切换到线程n
info threads| 查看线程
disassemble |反汇编（默认范围是选择帧的pc附近的函数）
info frame |选择堆栈帧
info args |显示函数参数和局部变量的内容
info reg（或者i r）| 查看地址
disas  |反汇编查看函数（包含地址信息）
i proc m （info proc mappings 的简写）| 核查零是不是有效地址

## 11.1. Bt

> 跟踪堆栈的信息: `bt [-a|-g|-r|-t|-T|-l|-e|-E|-f|-F|-o|-O] [-R ref] [-I ip] [-S sp] [pid | task]`

```
Bt 无参数则显示当前任务的堆栈信息
Bt –a 以任务为单位，显示每个任务的堆栈信息
Bt –t 显示当前任务的堆栈中所有的文本标识符
Bt –f 显示当前任务的所有堆栈数据，通过用来检查每个函数的参数传递
```

## 11.2. mod命令

> 用来加载调试符号，有时一些结构或者函数的符号信息不在调试版本内核里面，需要用`gcc -g`选项编译自己的模块，然后用mod命令加载里面的调试信息。这样`sym`和`whatis`命令就能正确解释我们自己模块里面自定义的结构等信息。

***

# 12. ffmpeg 转换jpg和png格式

`ffmpeg -i test.png test1.jpg`

# 13. 远程服务器使用ssh链接并且映射

在用户根目录的`.bashrc`添加：

1. 链接：
`alias sshTest='function _ssh() {  echo "提示信息"; ssh -p 端口 服务器用户名@IP; unset -f ssh; }; _ssh'`
2. 映射：
`alias sshreferenceTest='echo "提示信息";sshfs -p 端口 -o cache=yes,reconnect 服务器用户名@IP:/home/服务器映射目录 /home/user/本地映射目录'`

执行`source .bashrc`生效

> 如果服务器映射报错`bad mount point `/mnt/': Transport endpoint is not connected`

解决方法：
1. `sudo  umount  --all`（或者指定目录）
2. 重新mount，即`sudo mount --all`

***

# 14. linux下修改图片尺寸（jpg、png...）

1. `sudo apt-get install imagemagick`
2. `convert example.png -resize 200×100 example.png //按照原有比例缩放`
3. `convert example.png -resize 200×100! example.png`

# 15. 查看网络地址情况

`route -n`

# 16. 文件压缩、解压

+ zip文件：
`zip -r   a.zip  /dir`

直接unzip解压

+ tar.bz：
Linux下压缩比率较tgz大，即压缩后占用更小的空间，使得压缩包看起来更小。
但同时在压缩，解压的过程却是非常耗费CPU时间。

+ + 打包压缩格式，举例：
`tar -jcvf file.tar.bz2 dir #dir目录`

+ + 解压，举例：
`tar -jxvf file.tar.bz2`
`tar -jxvf file.tar.bz2 -C /temp`

***

# 17. ls 命令

Command|Notes
:-:|:-:
ls -a| 列出文件下所有的文件，包括以“.“开头的隐藏文件（Linux下文件隐藏文件是以.开头的，如果存在..代表存在着父目录）。
ls -l |列出文件的详细信息，如创建者，创建时间，文件的读写权限列表等等。
ls -F |在每一个文件的末尾加上一个字符说明该文件的类型。"@"表示符号链接、"|"表示FIFOS、"/"表示目录、"="表示套接字。
ls -s |在每个文件的后面打印出文件的大小。  size(大小)
ls -t |按时间进行文件的排序  Time(时间)
ls -A |列出除了"."和".."以外的文件。
ls -R |将目录下所有的子目录的文件都列出来，相当于我们编程中的“递归”实现
ls -L |列出文件的链接名。Link（链接）
ls -S |以文件的大小进行排序
ls -h|显示文件大小

# 18. firefox修改中文

1. 打开浏览器，在地址栏中输入`about:config`，然后按下回车。
2. 在列表中找到“general.useragent.locale”，然后双击，将内容改为"zh-CH"
3. 重启之后就会默认为中文了（注：如果想改回英文就改为:  en-US )

# 19. Ubuntu系统log路径`/var/log`

查看: `ls -tl`

# 20. hidl接口生成命令

`hidl-gen`是Android架构HIDL编译工具，可以手动将哈希加到`current.txt`中，也可以使用以下命令添加：
`hidl-gen -L hash -r .../interfaces -r android.hidl:...`