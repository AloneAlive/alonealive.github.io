---
layout: single
title:  Android Automotive Framework调试技巧
date:   2022-12-01 13:58:02 +0800
categories: android
tags: debug
toc: true
---

> Android系统调试技巧积累笔记，主要包含Android Framework，以及日常接触的git、adb、linux系统等调试技巧。

# 1. Android调试技巧

## 1.1. 查看socket链接状态：

```s
adb shell
# netstat -ap |grep 7777
tcp        0      0 0.0.0.0:7777            0.0.0.0:*               LISTEN      2596/test_service
tcp        0      0 localhost:7777          localhost:45634         ESTABLISHED 2596/test_service
tcp6       0      0 localhost:45634         localhost:7777          ESTABLISHED 5336/com.android.test
```

***

## 1.2. socket套接字

**socket的三次握手：**

1. 第一次握手：客户端需要发送一个syn j 包，试着去链接服务器端，于是客户端我们需要提供一个链接函数
2. 第二次握手：服务器端需要接收客户端发送过来的syn J+1 包，然后在发送ack包，所以我们需要有服务器端接受处理函数
3. 第三次握手：客户端的处理函数和服务器端的处理函数

三次握手只是一个数据传输的过程，但是，我们传输前需要一些准备工作，比如将创建一个套接字，收集一些计算机的资源，将一些资源绑定套接字里面，以及接受和发送数据的函数等等，这些功能接口在一起构成了socket的编程

**server服务端：**
1. socket()：创建socket
2. bind():绑定socket和端口号
3. listen():监听该端口号
4. accept():接收来自客户端的连接请求（阻塞等待，使用循环）
5. recv():从socket中读取字符（接收socket客户端的消息，可使用子线程控制多个连接）
6. close():关闭socket

**client客户端：**
1. socket()：创建socket
2. connect()：连接指定计算机的端口（**和服务端的accept()连接**）
3. send()：向socket中写入信息（**和服务端的recv()连接**）
4. close()：关闭socket

***

### 1.2.1. 网络字节顺序与本地字节顺序之间的转换函数

+ 参考：[htons(), ntohl(), ntohs()，htons()这4个函数](https://blog.csdn.net/zhuguorong11/article/details/52300680)

在C/C++写网络程序的时候，往往会遇到字节的网络顺序和主机顺序的问题。这是就可能用到`htons()`, `ntohl()`, `ntohs()`，`htons()`这4个函数。

**网络字节顺序与本地字节顺序之间的转换函数：**

+ htonl()--"Host to Network Long"
+ ntohl()--"Network to Host Long"
+ htons()--"Host to Network Short"
+ ntohs()--"Network to Host Short"

之所以需要这些函数是因为计算机数据表示存在两种字节顺序：`NBO与HBO`

+ 网络字节顺序NBO(Network Byte Order): 按从高到低的顺序存储，在网络上使用统一的网络字节顺序，可以避免兼容性问题。
+ 主机字节顺序(HBO，Host Byte Order): 不同的机器HBO不相同，与CPU设计有关，数据的顺序是由cpu决定的,而与操作系统无关。
+ + 如Intel x86结构下, short型数0x1234表示为34 12, int型数0x12345678表示为78 56 34 12  
+ + 如IBM power PC结构下, short型数0x1234表示为12 34, int型数0x12345678表示为12 34 56 78

***

## 1.3. Android签名

> Android的签名机制是一种Android保证系统安全的方式。Android中的签名就是对我们的应用或者系统加上特定的一些信息，防止被恶意篡改，而加上的这些特定信息就是签名。

### 1.3.1. 签名作用

1. 确保应用开发者身份的唯一性。因为Android应用包名可以随意替换，如果没有签名保证应用的唯一性，可能会造成其他的应用替换本身。
2. 保证应用在信息传输中的的完整性，签名对于包中的每个文件进行处理，以此确保包中内容不被替换或者被篡改。如果没有签名保护，则有可能被恶意程序盗取个人信息。

### 1.3.2. 签名的组成

> Android源码中的系统签名统一存放路径：build/target/product/security

+ `.pem类型文件`：在android对apk签名的时候，.pem这种文件就是一个X.509的数字证书，里面有用户的公钥等信息，是用来解密的。文件格式里面不仅可以存储数字证书，还能存各种key,这个可以公开，主要用于验证某个App或者其它的是否由相应的私钥签名。
+ `.pk8类型文件`：以.pk8为扩展名的文件，应该和PKCS 8是对应的，用来保存private key,并且这个私钥需要保密保存，不能公开。

Android系统签名路径下一共有5种key：
1. testkey：平台默认key。如果没有做特殊变更的话，系统编译默认使用该key。由于testkey是公开的，任何人都可以获取，所以存在一定的风险。如果项目由特殊需求，则一般使用 自己创建key作为系统默认key。 
2. platform：平台的核心应用签名之一，签名的apk是完成系统的核心功能。这些apk所在的进程UID是system。manifest节点中有添加sharedUserId="android.uid.system"。
3. shared：这个签名的apk可以和home/contacts进程共享数据。manifest节点中有添加android:sharedUserId="android.uid.shared"。 一般用在与tel相关的apk。
4. media:这个签名的apk是media/download的一部分。manifest节点中有添加android:sharedUserId="android.media"。 一般应用在media相关的一些apk。每个Apk包会在其对应的Android.mk中设置LOCAL_CERTIFICATE属性，指定其中一个密钥。（如果没有设置此变量,则默认使用testkey）
5. verity：一种特殊的系统签名。在系统编译时会对系统进行编译处理。需要单独生成

### 1.3.3. 签名的生成

> README中介绍了怎么通过内置工具生成和替换系统默认预置的各种key，以及各种key的简单使用实例


在Android 根目录下执行命令生成releasekey、platform、shared、media。以platform为例，其他key以相同方式生成

`development/tools/make_key platform'/C=US/ST=California/L=Mountain View/O=Android/OU=Android/CN=Android/emailAddress=android@android.com'`

**释义：**
+ 所在国家 (Country) 简称：C ，只能是国家字母缩写，如中国：CN
+ 所在省份 (State/Provice) 简称S 
+ 所在城市 (Locality) 简称L 
+ 单位名称 (Organization Name) ：简称O 
+ 部门名称（ Organizational Unit Name (eg, section) ）：简称OU
+ 公用名称 (Common Name) ：简称CN。代码签名证书申请单位名称或网站域名。
+ 邮箱（Email）

系统会提示让你输入生成该key所需要的密码，输入回车即可。如果设置key密码，则编译不通过。

### 1.3.4. 验证生成的key

生成的key存在于Android根目录下，把key移动到`/build/target/product/security`下并替换之前的key，同时需要使用OpenSSL的工具来验证一下生成的key是否正常。以releaseKey为例，执行以下命令：
`openssl x509 -noout -subject -issuer -in releaseKey.x509.pem`

输出：
```shell
subject= /C=CN/ST=NanJing/L=NanJing/O=13/OU=123/CN=123/emailAddress=test@123.com
issuer= /C=CN/ST=NanJing/L=NanJing/O=123/OU=13/CN=123/emailAddress=test@123.com
```

输出的内容与生成该key时输入的内容一致，则代表key验证成功

### 1.3.5. 生成verity_key

1. 编译verity：在android根目录下编译verity。`make generate_verity_key (mmm system/extras/verity/)`
2. 生成veritykey签名：`development/tools/make_key veritykey '/C=CN/ST=NanJing/L=NanJing /O=123/OU=13/CN=123/emailAddress=test@123.com'`
3. 执行以下命令生成.pk8 .pem文件：`out/host/linux-x86/bin/generate_verity_key -convert veritykey.x509.pem verity_key`

### 1.3.6. 签名的使用和配置

签名的使用分为两种情况：
1. 不区分user和debug版本，仅替换当前设备编译是所使用的key。
2. 分区user和debug版本，编译不同版本的时候使用不同的key。

#### 1.3.6.1. 不区分user和debug版本

1. 将android根目录下生成的各种.pk8和.pem文件copy到`build/target/product/security/`目录下，覆盖之前所有的默认key。将veritykey.pk8，veritykey.x509.pem，verity_key.pub重命名:verity.pk8，verity.x509.pem，verity_key
2. 修改android配置：

+ `/build/core/config.mk`中定义的变量：`DEFAULT_SYSTEM_DEV_CERTIFICATE := build/target/product/security/testkey`

**PS:**`PRODUCT_DEFAULT_DEV_CERTIFICATE`：mk文件中定义的`LOCAL_CERTIFICATE`变量，如果mk文件中定义了`LOCAL_CERTIFICATE`，编译则使用所定义的签名文件，如果没有定义LOCAL_CERTIFICATE 变量，则进入else流程

+ `/build/core/Makefile`中定义变量：`ifeq ($(DEFAULT_SYSTEM_DEV_CERTIFICATE),build/target/product/security/releasekey) BUILD_VERSION_TAGS += release-keys`

**PS:**这段code主要是判断在config.mk中定义的`DEFAULT_SYSTEM_DEV_CERTIFICATE`变量，并在编译的时候使用对应的build_key。在刷机后可以通过`getprop build.tags`来查看

+ 修改SELinux全新啊：`system/sepolicy/private/keys.conf system/sepolicy/prebuilts/api/{apilevel}/private/keys.conf`

***

#### 1.3.6.2. 分区user和debug版本

以testkey作为示例：
1. `device/project/`路径下新建一个用于存放新建key文件的文件夹security。仅用于存在key文件。后续步骤中会对路径做条件判断。
2. 在`device/project/product.mk`中进行条件判断：获取TARGET_BUILD_VARIANT，如果`TARGET_BUILD_VARIANT为userdebug/eng`，则使用`build/make/target/product/security/testkey`；否则使用我们自定义的文件夹中的key

### 1.3.7. Android设备判断系统签名key

通过以下命令验证打包编译好的系统使用的签名，这种方式只适用与不区分user和debug版本的第一种修改：

```shell
adb root;adb remount
adb shell
cd system
getprop | grep ro.build.tags

#可以看到一行 ro.build.tags=release-keys
```

如果要区分user和debug版本的key，则可以使用下面两个方法：
1. 在Android系统`adb install -r test.apk`安装进行判断
2. 从设备中随意pull一个apk出来。然后将后缀名.apk修改为.zip，接着进行解压操作，这样会在META-INF中看到CERT.RSA文件，通过`keytool -printcert -file \...\path\CERT.RSA`可以查看此apk的签名内容，如果签名内容与自己生成的key内容一致，则说明当前系统签名使用的是自己生成的key

***

### 1.3.8. 生成三方APP使用的签名文件

在三方App应用中，因为不用经过Android系统编译，所以如果没有签名文件的情况下用到特殊权限则无法安装使用。所以可以提供对应的.keystore文件供其使用

在自己生成的key所在的路径下：
1. 生成platform.pem文件：platform.pk8生成了`.pem`文件:`openssl pkcs8 -in platform.pk8 -inform DER -outform PEM -out platform.priv.pem -nocrypt`
2. 生成platform.p12文件: `openssl pkcs12 -export -in platform.x509.pem -out platform.p12 -inkey platform.pem -password pass:pwd -name name`，其中name为alias name，pwd为alias pwd
3. 生成platform.keystore：`keytool -importkeystore -deststorepass pwd-destkeystore ./platform.keystore -srckeystore ./platform.p12 -srcstoretype PKCS12 -srcstorepass pwd`

上面三个步骤可以在该路径下生成一个`platform.keystore`文件。该文件可用于apk签名使用。

***

## 1.4. Bootchart性能工具使用方式

> 参考[性能分析工具—bootchart工具使用](https://blog.csdn.net/c_z_w/article/details/83544687)

bootchart是一个用于linux启动过程性能分析的开源工具软件，在系统启动过程中自动收集CPU占用率、磁盘吞吐率、进程等信息，并以图形方式显示分析结果，可用作指导优化系统启动过程。

bootchart让用户可以很直观的查看系统启动的过程和各个过程耗费的时间，以便让用户能够分析启动过程，从而进行优化以提高启动时间。它由bootchartd服务和bootchart-render两部分组成，后者主要负责生成启动流程的分析结果图。

Android系统源码中有bootchart的实现，路径在`system/core/init/bootchart.cpp`中, bootchart通过内嵌在init进程中实现，在后台执行测量。**不过bootchart的测量时段是 init进程启动之后，不包含uboot和kernel的启动时间**

### 1.4.1. bootchart在android平台的使用步骤（复杂方式&old）

1. 使能调试设备的bootchart程序，进行设备必要的开机log:`adb shell 'touch /data/bootchart/enabled'`
2. 在reboot调试设备后，进入`/data/bootchart`目录，先行删除enabled, 再执行`tar -zcf bootchart.tgz *`， 接着`adb pull /data/bootchart/bootchart.tgz`到本地，拷贝到ubuntu
3. ubuntu机安装bootchart工具:`sudo apt-get install bootchart`和`sudo apt-get install pybootchartgui`
4. 生成bootchart图表：`bootchart bootchart.tg`

***

### 1.4.2. bootchart在android平台的使用步骤（方便方式&new）

1. 使能调试设备的bootchart程序，进行设备必要的开机log:`adb shell 'touch /data/bootchart/enabled'`
2. 在`adb reboot`调试设备后，ubuntu电脑连接调试设备，终端android根目录执行命令：`android$ ./system/core/init/grab-bootchart.sh`
3. bootchart的图标即生成在android根目录

***

### 1.4.3. 修改bootchart抓取的停止时间

> android高版本上不支持简单的设置方式调整bootchart的结束时间，只能在init.rc中修改，bootchart的启动和结束方式如下：

```shell
# Start bootcharting as soon as possible after the data partition is
 # mounted to collect more data.
 mkdir /data/bootchart 0755 shell shell encryption=Require
 bootchart start
 ...
on property:sys.boot_completed=1
bootchart stop
```

原生的逻辑中，在开机完成时，系统会设置`sys.boot_completed=1`，此时bootchart停止抓取信息；

我们可以更改stop的条件，自定义一个属性来实现停止，自己实现可控停止方式如下，在开机后，手动去设置这个属性值=1:

***

### 1.4.4. bootchart图形查看方式

整个图表以时间线为横轴，图标上方为CPU和磁盘的利用情况，下方是各进程的运行状态条，显示各个进程的开始时间与结束时间以及对CPU、I/O的利用情况，各个进程的运行时间以及CPU的使用情况，进而优化系统。

可以通过Laucher的启动完成时间判断开机完成完成时间，也就是开机动画结束的时间

***

## 1.5. 命令切横竖屏：

```shell
#切横屏:
su
settings put system user_rotation 3
settings put system accelerometer_rotation 0
#切竖屏:
settings put system accelerometer_rotation  1/2/ 3 随便改
```

***

## 1.6. Doze和App Standby模式下的Android应用适配

> 参考：[Doze和App Standby模式下的Android应用适配](https://www.jianshu.com/p/f044ce3f5913)

从Android6.0(API23)开始, Google为Android加入了两种省电特性，通过管理Android应用(以下简称应用)在非充电状态下的设备中的运行策略来达到延长用户的Android设备使用时间的目的。这两种特性存在一定的差别，Doze模式通过延缓应用在设备长时间待机状态下对于CPU和网络资源的使用来实现节能；而App Standby则是通过延缓最近未被使用的后台应用对于网络的请求来达到同样的目的。

Doze和App Standby在Android6.0及以上的Android设备中可以影响所有运行状态下的Android应用，无论这些应用的Target API是否是指定为API23。为了确保用户获得在不同Android版本下的应用体验一致性，开发者需要对应用在Doze及App Standby模式下做相应的适配。

## 1.7. fastboot解锁

短接后重新重启，然后执行`fastboot flashing unlock`

1. 短接下车机然后重启
2. 在window 命令行输入`fastboot unlock flashing`
3. 重启后执行接下来的adb root和remount操作

***

## 1.8. 查看make makefile（mk文件）执行的代码指令

`make -n`

例如：`make android -n`

## 1.9. 查看Android设备分区大小

```shell
adb shell
# 同时适用于linux系统
df -ah
#或者
df -h
```

***

## 1.10. kernel log命令

`adb shell cat proc/kmsg`

***

## 1.11. git push drafts

`git push origin HEAD:refs/drafts/master`

***

## 1.12. git push

`git push origin HEAD:refs/for/master`

或者使用`git remote -v`查看远程仓库push地址，替代origin

+ git commit如何修改默认编辑器为vim，修改`~/.gitconfig`：

```shell
[core]
    editor=vim
```

***

## 1.13. git补丁解决冲突方法

这种情况下可以使用下面的方法解决冲突：

1. 执行命令`git am xxxx.patch`尝试直接打入补丁。patch可能会报错并中断（注意，虽然命令停止执行了，但依然处于git am命令的运行环境中，可以通过`git status`命令查看到当前的状态）
2. 执行命令`git apply --reject xxxx.patch`自动合入patch中不冲突的代码改动，同时保留冲突的部分。这些存在冲突的改动内容会被单独存储到目标源文件的相应目录下，以后缀为`.rej`的文件进行保存。比如对`./test/someDeviceDriver.c`文件中的某些行合入代码改动失败，则会将这些发生冲突的行数及内容都保存在`./test/someDeviceDriver.c.rej`文件中。我们可以在执行`git am`命令的目录下执行`find  -name  *.rej`命令以查看所有存在冲突的源文件位置
3. 依据步骤2中生成的`*.rej`文件内容逐个手动解决冲突，然后删除这些`*.rej`文件。完成这一步骤的操作后，就可以继续执行`git am`的过程了
4. 执行命令`git status`查看当前改动过的以及新增的文件，确保没有多添加或少添加文件
5. 执行命令`git add .`将所有改动都添加到暂存区（注意，关键字add后有一个小数点.作为参数，表示当前路径）
6. 执行命令`git am --resolved`继续步骤1中被中断的patch合入操作。合入完成后，会有提示信息输出
7. 执行命令`git  log`确认合入状态

***

## 1.14. 报错...is locked by another process

**控制台输出：**

```shell
~$ sudo dpkg-reconfigure dash
debconf: DbDriver "config": /var/cache/debconf/config.dat is locked by another process: Resource temporarily unavailable
```

**解决方式：**

```shell
查询进程：
sudo lsof /var/cache/debconf/config.dat

杀进程：
sudo kill PID进程号
```

***

## 1.15. 报错HIDL库编译

```log
******************************************************
error: VNDK library: android.hardware.light@2.0's ABI has INCOMPATIBLE CHANGES Please check compatiblity report at : out/soong/.intermediates/hardware/interfaces/light/2.0/android.hardware.light@2.0/android_arm64_armv8-a_cortex-a53_vendor_shared/android.hardware.light@2.0.so.abidiff
******************************************************
 ---- Please update abi references by running platform/development/vndk/tools/header-checker/utils/create_reference_dumps.py -l android.hardware.light@2.0 ----

//执行该脚本后报错：
Traceback (most recent call last):
  File "development/vndk/tools/header-checker/utils/create_reference_dumps.py", line 224, in <module>
    main()
  File "development/vndk/tools/header-checker/utils/create_reference_dumps.py", line 210, in main
    targets = [Target(True, product), Target(False, product)]
  File "development/vndk/tools/header-checker/utils/create_reference_dumps.py", line 29, in __init__
    self.arch = build_vars[1]
IndexError: list index out of range
```

**解决方式：**

（1）环境判断：

```shell
执行: ls -l /bin/sh，查看用的是什么来解释执行Python脚本

$ ls -l /bin/sh
lrwxrwxrwx 1 root root 4 4月   2  2019 /bin/sh -> dash

如果用dash来解释执行Python脚本，得切换成bash
执行： sudo dpkg-reconfigure dash  选择no
```

**控制台输出：**

```log
Android$ development/vndk/tools/header-checker/utils/create_reference_dumps.py -l android.hardware.light@2.0
Removing reference dumps...
removing /home/workspace/Android/prebuilts/abi-dumps/vndk/28/32/arm_armv7-a-neon/source-based/android.hardware.light@2.0.so.lsdump.gz
removing /home/workspace/Android/prebuilts/abi-dumps/ndk/28/32/arm_armv7-a-neon/source-based/android.hardware.light@2.0.so.lsdump.gz
making libs for product: aosp_arm_ab
```

**再次执行：**

```s
//即自己编译的product name（out/target/product/test_product）
Android$ development/vndk/tools/header-checker/utils/create_reference_dumps.py -l android.hardware.light@2.0 -product test_product
Removing reference dumps...
removing /home//workspace/Android/prebuilts/abi-dumps/vndk/28/64/arm_armv8-a/source-based/android.hardware.light@2.0.so.lsdump.gz
removing /home//workspace/Android/prebuilts/abi-dumps/ndk/28/64/arm_armv8-a/source-based/android.hardware.light@2.0.so.lsdump.gz
removing /home//workspace/Android/prebuilts/abi-dumps/vndk/28/64/arm64_armv8-a/source-based/android.hardware.light@2.0.so.lsdump.gz
removing /home//workspace/Android/prebuilts/abi-dumps/ndk/28/64/arm64_armv8-a/source-based/android.hardware.light@2.0.so.lsdump.gz
making libs for product: test_product
Creating dumps for target_arch: arm and variant  armv8-a
Created abi dump at  /home//workspace/Android/prebuilts/abi-dumps/vndk/28/64/arm_armv8-a/source-based/android.hardware.light@2.0.so.lsdump.gz
Creating dumps for target_arch: arm64 and variant  armv8-a
Created abi dump at  /home//workspace/Android/prebuilts/abi-dumps/vndk/28/64/arm64_armv8-a/source-based/android.hardware.light@2.0.so.lsdump.gz

msg: Processed 2 libraries in  2.2664194742838544  minutes
```

***

## 1.16. Android源码改动不大时使用sync

```shell
# 项目代码根目录
adb root;adb remount
adb sync
adb shell sync
adb reboot
```

### 1.16.1. adb sync命令

+ 命令意思：同步更新/data/或/system/下的数据（也会包含vendor）
+ 命令用法：`adb sync [directory]`

如果不指定目录，将同步更新/data/和/system/

### 1.16.2. adb shell sync

1. 在shell中执行
2. 将内存缓冲区中的数据写入到磁盘

**PS：**为了避免对硬盘的频繁读写，数据一般存放在缓冲区。什么情况下会把缓冲区的数据写入到磁盘中：
1. 通过调用fflush函数刷新缓冲区
2. 缓冲区已满（8k）
3. 正常关闭文件，如下：
+ 调用fclose
+ 主函数调用return
+ 调用exit函数

***

## 1.17. 修改selinux后make编译

1. `make selinux_policy`：这条命令可以编译所有selinux⽂件，50s左右即可完成⼀次make
2. `make ⽂件名`：**例如：**

```shell
make plat_file_contexts -j9;
make plat_sepolicy.cil -j9;
make vendor_sepolicy.cil -j9;
make vendor_property_contexts -j9;
```

### 1.17.1. 如何同步selinux修改到device

如果使⽤make出来的selinux⽂件，建议`adb sync`命令同步到device中

或以下命令：

```shell
adb push out/target/product/product/vendor/etc/selinux/* vendor/etc/selinux/*
adb push out/target/product/product/system/etc/selinux/* system/etc/selinux/*
```

***

## 1.18. carservice的jar包

```s
/out/target/common/obj/JAVA_LIBRARIES/android.car_intermediates$ ll
total 1988
drwxr-xr-x   2 user domain users   4096 Nov 26 16:34 ./
drwxr-xr-x 161 user domain users  12288 Nov 23 10:13 ../
-rw-r--r--   1 user domain users 578273 Nov 26 16:16 classes-header.jar
-rw-r--r--   1 user domain users 782057 Nov 26 16:16 classes.jar   ---------> 该文件作为app导入的jar包
-rw-r--r--   1 user domain users      0 Nov 12 11:24 exported-sdk-libs
-rw-r--r--   1 user domain users 648154 Nov 26 16:34 javalib.jar
-rw-r--r--   1 user domain users     14 Nov 12 11:24 link_type
```

***

## 1.19. UpdateEngine升级模块学习博客

**Upgrade OTA升级相关文档：**

**系列文章：**
+ [系列-Android A/B System OTA分析和Update模块](https://blog.csdn.net/guyongqiangx/article/details/71334889)
+ [系列-Android OTA升级原理和流程分析](https://blog.csdn.net/twk121109281/article/details/90715730)
+ [系列-android编译系统分析（五）system.img的生成过程](https://blog.csdn.net/u011913612/article/details/52503318)
+ [android P OTA 初探 —— 1、OTA简单介绍](https://blog.csdn.net/liyuchong2537631/article/details/97516299)
+ [android P OTA初探 —— 2、基于块（Block）的OTA：Target 包的制作流程](https://blog.csdn.net/liyuchong2537631/article/details/97517850?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-13&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-13)
+ [android P OTA初探 —— 3、基于块（Block）的OTA：升级包的制作流程](https://blog.csdn.net/liyuchong2537631/article/details/97528659?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-3&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-3)


**单篇文章：**
+ [Google原生OTA升级说明文档](https://source.android.google.cn/devices/tech/ota)
+ [A/B 升级update_engine分析-整体模块](https://blog.csdn.net/Android_2016/article/details/101554104)
+ [Android recovery分析（一）---全量升级包的编译流程](https://blog.csdn.net/mrcc_yy/article/details/51615540)
+ [Android Recovery OTA升级（一）—— make otapackage](https://blog.csdn.net/wzy_1988/article/details/47056035)
+ [Android Recovery 源码解析和界面定制](https://blog.csdn.net/austindev/article/details/55213444)
+ [Android OTA 差分包升级](https://www.cnblogs.com/MMLoveMeMM/articles/4086796.html)
+ [update_engine简介](https://www.cnblogs.com/startkey/p/10553672.html)
+ [Android P update_engine分析（六）-- PostinstallRunnerAction的工作](https://www.csdn.net/tags/NtDagg5sMzE1ODAtYmxvZwO0O0OO0O0O.html)
+ [A/B 升级update_engine分析-Action流程](https://blog.csdn.net/Android_2016/article/details/102912469)
+ [Android Update Engine分析（七） DownloadAction之FileWriter](https://blog.csdn.net/guyongqiangx/article/details/82805813)

**独立文章：**

+ [汽车OTA介绍](https://zhuanlan.zhihu.com/p/86449761)
+ [什么是T-BOX？](https://zhuanlan.zhihu.com/p/63560871)
+ [android ota升级理论1](https://zhuanlan.zhihu.com/p/69390321)

***

## 1.20. update_engine升级包的hash值计算

> 升级包解压后在payload_properties.txt文件可以看到payload.bin和metadata的文件大小和hash值
> 
> 该hash值是通过sha256sum计算后，再base64编码成字符串
> 
> 参考：[Android Update Engine分析（十） 生成 payload 和 metadata 的哈希](https://blog.csdn.net/guyongqiangx/article/details/122393172)

**实际操作：**

```shell
$ cat ../../../payload_properties.txt 
#payload.bin升级包数据
FILE_HASH=Kw0egs8dY9Qm1sULkpkAPF+0PiKx14Uo0fHGZO3vNAk=
FILE_SIZE=2515281637
#压缩后的 manifest 数据
METADATA_HASH=z+5Qz8UIBrhCtMDgo+5IG4/rQqFUqmN/FMFyIWhaIEU=
METADATA_SIZE=179557

$ ls -l payload.bin 
#文件大小对应
-rw-r--r-- 1 user domain users 2515281637 Jan  1  2009 payload.bin
$ sha256sum payload.bin | awk '{print $1}' | xxd -r -ps | base64
#计算后的文件hash值对应
Kw0egs8dY9Qm1sULkpkAPF+0PiKx14Uo0fHGZO3vNAk=

#计算后的metadata hash值对应
$ dd if=payload.bin bs=1 count=179557 2>/dev/null | sha256sum | awk '{print $1}' | xxd -r -ps | base64 
z+5Qz8UIBrhCtMDgo+5IG4/rQqFUqmN/FMFyIWhaIEU=
```

***

## 1.21. 编译升级包出现报错LKRRPART failed：Device or resource busy

**输入以下命令：**
```shell
sudo losetup -d /dev/loop0
sudo losetup -d /dev/loop1
sudo losetup -d /dev/loop2
sudo losetup -d /dev/loop3
```

***

## 1.22. 原生升级系统的Demog APK

adb shell am start com.android.car.systemupdater/.SystemUpdaterActivity

## 1.23. 原生carsystemUI apk

```s
/out/target/product/product/system$ find . -iname "*systemui*"
./product/priv-app/CarSystemUI
./product/priv-app/CarSystemUI/CarSystemUI.apk
./product/etc/permissions/com.android.systemui.xml
```

***

## 1.24. update_engine升级信息获取

```shell
adb root
adb pull /data/misc/update_engine_log
adb pull /data/misc/update_engine/
```

```shell
adb shell
/data/misc/update_engine/prefs # ls -thl
total 38K
-rw------- 1 root root   1 2021-01-01 00:11 total-bytes-downloaded
-rw------- 1 root root  17 2021-01-01 00:11 system-updated-marker
-rw------- 1 root root   1 2021-01-01 00:11 delta-update-failures
-rw------- 1 root root  36 2021-01-01 00:11 update-completed-on-boot-id
-rw------- 1 root root   4 2021-01-01 00:11 post-install-succeeded
-rw------- 1 root root   4 2021-01-01 00:09 verity-written
-rw------- 1 root root   1 2021-01-01 00:08 update-state-next-data-length
-rw------- 1 root root   9 2021-01-01 00:08 update-state-next-data-offset
-rw------- 1 root root   4 2021-01-01 00:08 update-state-next-operation
-rw------- 1 root root 112 2021-01-01 00:08 update-state-sha-256-context
-rw------- 1 root root 264 2021-01-01 00:08 update-state-signature-blob
-rw------- 1 root root 112 2021-01-01 00:08 update-state-signed-sha-256-context
-rw------- 1 root root   6 2021-01-01 00:02 manifest-metadata-size
-rw------- 1 root root   3 2021-01-01 00:02 manifest-signature-size
-rw------- 1 root root   4 2021-01-01 00:02 dynamic-partition-metadata-updated
-rw------- 1 root root   1 2021-01-01 00:02 resumed-update-failures
-rw------- 1 root root  88 2021-01-01 00:02 update-check-response-hash
-rw------- 1 root root  36 1970-01-01 08:00 boot-id
-rw------- 1 root root  24 1970-01-01 08:00 previous-version
:/data/misc/update_engine/prefs # cat boot-id
e2ecfeb4-d097-449c-8c7b-e8aee5067cc6

:/data/misc/update_engine/prefs # cat previous-version
eng.user.20211201.031118

:/data/misc/update_engine/prefs # cat update-check-response-hash
LBhFc9jN99sOu9/VKy7COfkBXtXzlDt5fHdx1kQyU3c=vsNI/ZuKzDY0uOuXZIOjr42UHrzJ86isJSEtAwlFf9k=
```

***

## 1.25. update_engine错误码errorcode

```cpp
//android/system/update_engine/common/error_code.h
// Action exit codes.
enum class ErrorCode : int {
  kSuccess = 0,   //升级成功
  kError = 1,  //升级失败
  kOmahaRequestError = 2, //请求action错误（action机制用于控制升级每个步骤）
  kOmahaResponseHandlerError = 3,   //返回handler action错误
  kFilesystemCopierError = 4,  //文件系统拷贝错误
  kPostinstallRunnerError = 5,   //预编译运行步骤错误（PostinstallRunner是一个升级步骤）
  kPayloadMismatchedType = 6,    //NOT NEED
  kInstallDeviceOpenError = 7, //安装设备打开错误
  kKernelDeviceOpenError = 8, //内核设备打开错误
  kDownloadTransferError = 9, //下载传输错误
  kPayloadHashMismatchError = 10, //升级包hash未匹配错误
  kPayloadSizeMismatchError = 11,   //升级包size未匹配错误
  kDownloadPayloadVerificationError = 12, //下载过程升级包校验错误
  kDownloadNewPartitionInfoError = 13, //下载过程新分区信息错误
  kDownloadWriteError = 14,   //下载过程数据写入错误
  kNewRootfsVerificationError = 15, //升级分区hash校验失败
  kNewKernelVerificationError = 16, //升级kernel校验失败
  kSignedDeltaPayloadExpectedError = 17,  //NOT NEED
  kDownloadPayloadPubKeyVerificationError = 18, //下载过程升级包public key公钥校验错误
  kPostinstallBootedFromFirmwareB = 19,   //NOT NEED
  kDownloadStateInitializationError = 20, //下载状态初始化错误
  kDownloadInvalidMetadataMagicString = 21, //NOT NEED
  kDownloadSignatureMissingInManifest = 22,  //下载过程manifest缺少签名错误
  kDownloadManifestParseError = 23, //下载过程manifest分析错误
  kDownloadMetadataSignatureError = 24,   //下载过程元数据签名错误
  kDownloadMetadataSignatureVerificationError = 25,   //下载过程元数据签名校验错误
  kDownloadMetadataSignatureMismatch = 26,   //下载过程元数据签名不匹配错误
  kDownloadOperationHashVerificationError = 27, //下载过程操作hash校验错误
  kDownloadOperationExecutionError = 28,  //下载过程操作执行错误
  kDownloadOperationHashMismatch = 29, //下载过程操作hash不匹配错误
  kOmahaRequestEmptyResponseError = 30,   //请求action无返回错误
  kOmahaRequestXMLParseError = 31,  //请求action分析xml错误
  kDownloadInvalidMetadataSize = 32,   //下载过程非法元数据大小
  kDownloadInvalidMetadataSignature = 33, //下载过程非法元数据签名
  kOmahaResponseInvalid = 34, //返回action非法错误
  kOmahaUpdateIgnoredPerPolicy = 35,   //NOT NEED（含义是接收已回滚版本，忽略此次升级）
  kOmahaUpdateDeferredPerPolicy = 36,  //NOT NEED（含义是因更新策略延迟，忽略此次升级）
  kOmahaErrorInHTTPResponse = 37,   //HTTP返回错误
  kDownloadOperationHashMissingError = 38,   //下载过程操作时缺失hash错误
  kDownloadMetadataSignatureMissingError = 39,  //下载过程元数据签名缺失错误
  kOmahaUpdateDeferredForBackoff = 40,    //NOT NEED（含义是忽略本次升级）
  kPostinstallPowerwashError = 41,  //NOT NEED（回滚报错，该版本已去除版本回滚限制）
  kUpdateCanceledByChannelChange = 42, //通道变化升级取消
  kPostinstallFirmwareRONotUpdatable = 43, //NOT NEED(需要升级固件firmware时才会取消，因为无法从FW B分区启动到FW A分区)
  kUnsupportedMajorPayloadVersion = 44, //获取manifest偏移量错误
  kUnsupportedMinorPayloadVersion = 45,   //未manifest可支持更小版本错误
  kOmahaRequestXMLHasEntityDecl = 46, //请求action xml hash非法错误
  kFilesystemVerifierError = 47, //文件系统校验错误（FilesystemVerifier是一个升级步骤）
  kUserCanceled = 48, //用户取消
  kNonCriticalUpdateInOOBE = 49, //NOT NEED（Ignoring a non-critical Omaha update before OOBE completion.）
  kOmahaUpdateIgnoredOverCellular = 50,   //NOT NEED（未设置设备策略，因此用户首选项需要覆盖是否允许通过蜂窝网络进行更新）
  kPayloadTimestampError = 51, //升级包时间戳错误 （payload.bin是OTA镜像打包文件）
  kUpdatedButNotActive = 52,  //升级分区非action状态错误
  kNoUpdate = 53, //无升级（There are no updates. Aborting.）
  kRollbackNotPossible = 54,  //NOT NEED
  kFirstActiveOmahaPingSentPersistenceError = 55,  //NOT NEED（用于旧设备的Omaha校验）
  kVerityCalculationError = 56,  //校验计算错误（在FilesystemVerifier步骤中进行分区校验时）
```

***

## 1.26. Android代码架构释义

> [Android系统源代码目录与系统目录](https://www.jianshu.com/p/d92ad2e4b529)
> 
> [Android源码及系统目录结构分析](https://blog.csdn.net/zxc024000/article/details/110825669)
> 
> [Ubuntu16.04编译Android源码系列](https://blog.csdn.net/u012195899/article/details/82078384)

```shell
- art/  # Android runtime运行环境, java虚拟机art源码
- bionic/ # C库
- bootable/	# 启动引导和恢复系统相关代码（bootloader和recovery）
   - bootloader
      - dramk_2712/	
   	- lk/	 # liitle kernel，负责引导Linux的⼀个迷你⼩系统，启动顺序位于fastboot/uboot之后，Linux kernel之前，系统开机之后是进Android还是进recovery，即是这⾥负责
	   - lk_ext_mod/
   - recovery # 负责升级、恢复出⼚等⼀些⽐较重要⼜不是很复杂的⼯作，一般属于升级update子模块

- bootstrap.bash	
- build/	# 存放系统编译规则等基础开发包配置
   - blueprint # .bp的全程，由Go语⾔编写，是⽣成、解析Android.bp的⼯具，是Soong的⼀部分。Soong则是专为Android编译⽽设计的⼯具，Blueprint只是解析⽂件的形式，⽽Soong则解释内容的含义
   - kati # 基于Makefile来⽣成ninja.build的⼩项⽬。主要⽤于把Makefile转成成ninja file，⾃⾝没有编译能⼒，转换后使⽤Ninja编译
   - make
   - soong # Soong（go语言写的项目）构建系统是在Android 7中引⼊的，旨在取代Make。它利⽤Kati GNU Make克隆⼯具和Ninja构建系统组件来加速 Android 的构建；Soong在编译时使⽤，解析Android.bp，将之转化为Ninja⽂件

- compatibility/	# 兼容性计划文档目录，一些markdown文档
- cts/	# cts测试用例源码，但是一般cts跑测是从官网下载官方更新的cts测试包，如果跑测出现case faile，可以根据log在这边阅读源码分析
- dalvik/	# java dalvik虚拟机
- developers/	# demo源码和开发者参考文档
   - demo
   - samples
- development/	# 应用程序开发相关的实例/模板/工具
   - apps  # 开发者选项等app
   - tools

- device/	# 芯片厂商定制代码，包含AOSP自带的多家芯片厂以及芯片平台的代码，android O引入treble后，该目录下大部分关于厂商定制的源码移到vendor目录下
   - mediatek_projects # 此目录存放芯片平台平台下，几种不同的变种芯片项目编译定制化内容

- external/	# 开源第三方组件模块，存放着⼤量Google在创造、更新Android过程中，因实现某些功能⽽引⼊的开源第三⽅库，有些库是做了多系统适配（例如适配Android、Win、Linux），有些是only for Android
   - aac/ # aac解码库，AAC是高级音频编码（Advanced Audio Coding）
   - bsdiff/ # diff工具

- frameworks/	# framework层，应用程序框架层
   - av
      - camera # cameram模块
      - cmds 
         - screenrecord/	# 录屏工具源码
	      - stagefright/	   # 一个多媒体框架
      - media # audio、media源码
      - packages # 只有一个MediaComponents apk源码
      - services # audioflinger、cameraservice、mediacodec等核心源码
      - soundtrigger # audio模块的语音识别唤醒相关
   - base
      - api #Android对外暴露的API整体存放位置，make update-api命令更新的就是这里的文件
      - cmds # 命令/工具源码路径，例如am、pm、screencap等
         - bootanimation/	# 开机动画
      - core # android java和JNI核心库源码，例如framework.jar,hwbinder.jar,libandroid_runtime等
         - jni # libandroid_runtime
      - data # Android系统中字体、效果音等资源文件存放目录
      - graphics # graphic基础补充库，属于framework.jar
      - libs # display模块的hwui库位于这个目录下，以及其他的几个模块
      - media # media部分的java实现和jni实现源码，java部分属于framework.jar
      - opengl # opengl java api，隶属于framework.jar
      - packages # aosp core app源码路径，有DocumentsUI、Keyguard、MtpDocumentsProvider、SettingsProvide、SystemUI、Wallpaper等
      - services # service.jar源码，systemServer中启动的多数java service服务代码所在地
      - telecomm/	# tele模块，属于framework.jar
      - telephony/
      - wifi # wifi java api源码目录，属于framework.jar
   - native
      - cmds # bugreport、dumpsys等Android debug工具的c实现源码，以及serviceManager
      - libs # binder的Android中间件层实现源码，及libgui、libui、libsensor等中间件库源码
      - opengl # OpenGL的c/c++实现源码
      - services # 一些Android核心C进程的C/C++源码，例如surfaceFlinger、inputflinger、sensorservice等

- gerrit_config.sh
- hardware/	 # 硬件适配层hal层
   - interfaces/  # Android aosp定义的hidl interface实现，hal service实现源码
   - libhardware
      - moudles
         - audio
         - camera
         - input
   - mediatek  # 芯片平台 HAL实现代码，上层对接Google HIDL interface

- kernel/ # aosp的kernel适配目录
- kernel-4.9/	# kernel层源码，Android P之前使用，Q和R是4.14
- libcore/	# Java api核心库实现源码，dalvik、art部分java实现代码也在这里
   - dalvik
- libnativehelper/	# native层helper，即JNJ相关cpp源码
- Makefile	# 全局makefile
- packages/	# 大部分android apk源码
   - apps
      - Calendar #日历应用
      - Camera2/ # 官方camera
      - car
      - Email/
      - Gallery # 图库
      - SoundRecorder   # 录音机
   - providers # 存放着一堆provider的源码，media模块使用的MediaProvider，ContactsProvider等
   - services  # java service源码位置
      - car  # carservice源码

- pdk/	# platform development kit，平台开发工具
- platform_testing/	# 平台一些测试相关的源码
- prebuilts/	# 预编译资源
   - go # go语言环境，比如用于支持soong
- sdk/	# sdk工具源码
- system/	# 底层文件系统库、应用、组件，特殊功能模块
   - bt # 蓝牙相关
   - connectivity # wifi相关库
   - core # Android C/C++部分核心组件源码，有相当一部分是common类的组件
      - adb # adb进程源码
      - fastboot  # fastboot命令工具源码
      - healthd   # 电池服务守护进程源码
      - init      # init进程实现源码
      - libcutils # 工具库源码，例如property_set
      - ibion # ion内存管理工具类源码
      - liblog # ALOGX等相关函数实现源码
      - libusbhost # usbhost实现库
      - logcat # logcat实现源码
      - logd # 负责输出log的logd守护进程源码
      - sdcard # 外置存储卡挂载时会使用到的工具类
   - hwservicemanager   # hwservicemanager的实现源码，负责管理hidl service的，类似serviceManager
   - libhidl # hidl功能的c/c++层实现源码
   - libhwbinder # hwbinder功能的实现源码
   - sepolicy # aosp selinux配置文件所在目录
   - vold # 用于磁盘管理的守护进程

- test/	# vts测试源码
   - vts
- toolchain/ # Android工具链
- tools/	# android一些工具的源码，例如apk签名
   - apksig
- vendor/	# 厂商定制模块（芯片厂商以及我司定制化功能的实现代码）
   - mediatek  # 
      - external/	
	   - hardware/	
	   - kernel_modules/	
	   - packages/	# 芯片平台方案的私有app源码
      - proprietary/
```

***

## 1.27. Android编译三步命令解析

### 1.27.1. source build/envsetup.sh

1.source执行device和vendor⽬录下⾯⼀堆vendorsetup.sh⽂件，该shell脚本实就是调⽤add_lunch_combo函数，注册了对应的product

**PS：**如果我们需要⾃定义新增⼀个product，那么第⼀步，我们应该：在vendor⽬录或device⽬录下（这⾥推荐在vendor⽬录下），新增⼀个vendorsetup.sh⽂件，并且调⽤add_lunch_combo函数将我们期望增加的product name传递进去。

2.在执⾏完`source build/envsetup.sh`命令后，定义⼀堆shell到当前的⽤⼾环境中，此时可以输⼊`hmm`命令看到这些扩展shell命令，或者执行`m help`

可以在`build/envsetup.sh`文件头看到以下的注释：

|命令|解释|
|:-:|:-:|
|lunch|lunch <product_name>-<build_variant>选择<product_name>作为要构建的产品，<build_variant>作为要构建的变体，并将这些选择存储在环境中，以便后续调⽤“m”等读取|
|tapas| 交互⽅式：tapas [..] (arm/x86/mips/arm64/x86_64/mips64)(eng/userdebug/user)|
|croot |将⽬录更改到树的顶部或其⼦⽬录|
|m |编译整个源码，可以不⽤切换到根⽬录|
|mm |编译当前⽬录下的源码，不包含他们的依赖模块|
|mma |编译当前⽬录下的源码，包含他们的依赖模块|
|mmm |编译指定⽬录下的所有模块，不包含他们的依赖模块 例如：mmm dir/:target1,target2|
|mmma |编译指定⽬录下的所模块，包含他们的依赖模块|
|provision|具有所有必需分区的闪存设备。选项将传递给fastboot|
|cgrep |对系统本地所有的C/C++ ⽂件执⾏grep命令|
|ggrep |对系统本地所有的Gradle⽂件执⾏grep命令|
|jgrep |对系统本地所有的Java⽂件执⾏grep命令|
|resgrep| 对系统本地所有的res⽬录下的xml⽂件执⾏grep命令|
|mangrep |对系统本地所有的AndroidManifest.xml⽂件执⾏grep命令|
|mgrep |对系统本地所有的Makefiles⽂件执⾏grep命令|
|sepgrep| 对系统本地所有的sepolicy⽂件执⾏grep命令|
|sgrep |对系统本地所有的source⽂件执⾏grep命令|
|godir| 根据godir后的参数⽂件名在整个⽬录下查找，并且切换⽬录|

***

#### 1.27.1.1. user/userdebug/eng区别

`user/userdebug/eng`标签指的是Android.mk编译脚本⾥对应的属性`LOCAL_MODULE_TAGS`，LOCAL_MODULE_TAGS的取值有`user、debug、eng、tests、optional`五个。

optional的意思是该模块不主动参与编译，需要开发者⾃⾏在对应的mk⽂件中include这个模块，就是在device.mk中的`PRODUCT_PACKAGES`变量中引⼊该模块

+ user版

```
1. root权限关闭，默认情况下⽆法adb root，⽆法adb remount，发货版本均为user版；
2. 仅标签为user的模块会参与编译；
3. app及jar包，及java源码的代码混淆被开启（除⾮显式指定关闭的模块会关闭）；
4. 不包含D级别log输出
```

+ userdebug版

```
1. root权限开启，可remount，开发常⽤版本；
2. 标签为user、userdebug的模块会参与编译
```
+ eng版

```
1. root权限开启；
2. 标签为user、userdebug、eng的模块参与编译
```

#### 1.27.1.2. adb remount

`adb remount`将`/system`部分置于可写入的模式，默认情况下`/system`部分是只读模式的。这个命令只适用于已被root的设备。

在将文件push到`/system`文件夹之前，必须先输入命令`adb remount`。

`adb remount`的作用相当于`adb shell mount -o rw,remount,rw /system`

`adb remount`重新挂载system分区，实现对system分区重新挂载，重新挂载的时候将修改分区的属性，常见的修改参数为分区的读写。

使用该命令主要是因为android系统的system分区在启动之后是只读分区，但在开发过程中需要对system分区进行修改，则需重新挂载成读写模式

***


## 1.28. 内存泄露

> 参考：[内存泄漏问题初探](https://zhuanlan.zhihu.com/p/358838529)
> 
> 参考：[性能问题分析方法（1） — RAM](https://blog.csdn.net/yun_hen/article/details/116604034)
> 
> 参考：[Android Display/Graphics调试技巧（六月份更新）](https://wizzie.top/Blog/2021/06/07/2021/210607_android_debug3/)

1.物理内存泄漏

一般物理内存泄漏会导致`kernel oom-killer` 杀进程内存

日志中会打印具体哪个进程`invoked oom-killer`，以及申请内存的堆栈，可以据此分析。

2.native 进程内存泄漏

从`event log`中可以看到进程的内存使用信息。

一般通过`am_pss`中能看个各个进程内存使用情况，如果某个native进程的am_pss中uss过高，大概率是它引起内存泄漏

```log
am_pss: Pid, UID, ProcessName, Pss, Uss
am_pss (Pid|1|5),(UID|1|5),(Process Name|3),(Pss|2|2),(Uss|2|2),(SwapPss|2|2),(Rss|2|2),(StatType|1|5),(ProcState|1|5),(TimeToCollect|2|2)

am_meminfo: Cached,Free,Zram,Kernel,Native
am_low_memory: NumProcesses

//内存含义
vss (virtual set size) 虚拟内存，从进程地址空间统计，包括未实际申请到物理内存的部分。
rss (resident set size) 实际物理内存+共享库。共享库如果映射多个进程，不均摊。
pss ( proportional set size) 物理内存+均摊的共享内存。
uss (unique set size) 独占物理内存，不包含共享库部分（进程所占用的私有内存（内存泄漏的最佳观察数据）不包含so库占用内存的）

//例如
Line 82266: 01-01 08:24:27.599  2697  2798 I am_pss  : [3541,1000,vendor.projection.androidauto,51497984,47623168,0,133905408,0,5,7]
```

3.虚拟地址内存泄漏

`vss oom`，多发生在32位应用。虚拟地址空间不足，无法申请到 vma，所以申请内存失败。
一般只有发生泄漏的应用会崩溃，物理内存情况可能使用并不多，虚拟内存可能接近 4G(32位)。
一般需要 smaps/maps 信息做进一步分析，确认哪种类型的 vma 占比较多，那么大概率是它泄漏（比如 libc, egl，ion，等等）

4.ion内存泄漏

ion是android中特有的内存分配器，一般在camera，图像中使用较多。

通过 ion 的 ioctl 接口，可以申请不同类型的内存，可以为指定进程预留，比如 物理地址空间连续（cma），为相机预留。

ion 申请的内存需要主动释放，不释放会存在泄漏。每一个 heap 中申请的都对应一个 ion_buffer，我们可以统计 ion buffer 中某个进程占比多少（高通支持）。

如果 ion 的总量特别大（比如 4/8G），那么大概率是 ion 泄漏。再通过每个进程的信息，确定到是哪个进程申请导致的泄漏。

***

### 1.28.1. 查看进程占用内存情况

`adb shell procrank`

### 1.28.2. dump信息

+ `adb shell dumpsys cpuinfo > cpuinfo.txt`
+ `adb shell dumpsys meminfo > meminfo.txt`
+ `top -d 3`

**结果显示：**

```shell
Tasks: 227 total,   1 running, 226 sleeping,   0 stopped,   0 zombie
  Mem:      2.8G total,      2.1G used,      733M free,      4.0M buffers
 Swap:      1.4G total,       38M used,      1.3G free,      323M cached
400%cpu  42%user   0%nice 111%sys 233%idle   0%iow   8%irq   6%sirq   0%host
  PID USER         PR  NI VIRT  RES  SHR S[%CPU] %MEM     TIME+ ARGS
 4279 shell        20   0  35M 9.0M 2.4M R 55.5   0.3   0:00.15 top -d 3
 2316 system       -3 -20 165M  16M 5.0M S 27.7   0.5  17:25.76 android.hardware.graphics.composer@2.3-service
 2314 system       -2  -8 340M  30M  15M S 25.0   1.0  16:40.84 surfaceflinger
```

***

## 1.29. 查看进程优先级

1.`ps -A|grep 进程名`
2.查看进程优先级:

```shell
adb shell
# 如果该进程其值为0。0表示该进程是正在前台运行
cat proc/917/oom_adj
```

***

## 1.30. 查看系统服务

```s
$ service list|grep Update
60      updatelock: [android.os.IUpdateLock]
61      system_update: [android.os.ISystemUpdateManager]
105     webviewupdate: [android.webkit.IWebViewUpdateService]
147     android.os.UpdateEngineService: []
```

***

## 1.31. 解析crash地址

在对应包的symbols下查找库文件，使用addr2line命令：

```shell
symbols/vendor$ addr2line -f -e bin/test_service 000000000000ac40
_ZN12UpdateSocket9handleMsgEPv
vendor/..../test.cpp:87
```

***

## 1.32. 编码过程中单词常用的缩写方式（转载）

+ 参考：[编码过程中单词常用的缩写方式（转载）](https://blog.csdn.net/weixin_41178745/article/details/109765561)

***

## 1.33. 清除应用数据及查看版本

+ 清除应用数据:`adb shell pm clear com.test.upgrade`
+ 查看应用版本号：`dumpsys package com.test.upgrade |grep version`

***

## 1.34. Android11.0(R) 添加新分区

> 参考：[Android11.0(R) 添加新分区](https://cpu52.com/archives/373.html)

## 1.35. 系统编译中LOCAL_CFLAGS宏用法

> 参考：[Android.bp 添加宏开关](https://blog.csdn.net/lqktz/article/details/84959151)
> 
> 参考源码：`hardware/interfaces/configstore/1.1/default/surfaceflinger.mk`

`LOCAL_CFLAGS += -DXXX`:相当于在所有源文件中增加一个宏定义`#define XXX`

**例如：**

在Android.mk中增加：
```shell
ifeq ($(PRODUCT_MODEL),XXX_A)
LOCAL_CFLAGS += -DBUILD_MODEL
endif
```

即能在所编译的cpp文件中使用：
```shell
#ifdef BUILD_MODEL
....
#endif
```

***

## 1.36. android只编译system或者vendor

+ 编译system：`make snod`
+ 编译vendor：`make vendorimage`

***

## 1.37. repo切换分支

`repo forall -c git checkout Branch`:切换分支

***

## 1.38. Android 编译不生成odex文件（编译时不优化）

+ 参考：[ODEX优化和配置](https://www.showdoc.com.cn/2701?page_id=735330408300734)
+ [Android 编译不生成odex文件（编译时不优化）](https://blog.csdn.net/FeiPeng_/article/details/122061946)

在Android.mk配置：
`LOCAL_DEX_PREOPT := false`

***

## 1.39. emmc分区检查

+ [Linux e2fsck修复下受损的硬盘文件命令详解](https://www.csdn.net/tags/NtjaAgxsNTk5OTItYmxvZwO0O0OO0O0O.html)
+ [Linux 磁盘维护 : e2fsck 命令详解](https://blog.csdn.net/yexiangCSDN/article/details/83181885)

### 1.39.1. 使用e2fsck命令

```shell
8|console:/dev/block/by-name # e2fsck -d vendor_a
e2fsck 1.44.4 (18-Aug-2018)
vendor: clean, 748/32768 files, 120659/131072 blocks
console:/dev/block/by-name #

console:/dev/block/by-name # e2fsck -a vendor_a
vendor: clean, 748/32768 files, 120659/131072 blocks
console:/dev/block/by-name #

console:/dev/block/by-name # e2fsck -c vendor_a
e2fsck 1.44.4 (18-Aug-2018)
sh: badblocks: inaccessible or not found
vendor: Updating bad block inode.
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information

vendor: ***** FILE SYSTEM WAS MODIFIED *****
vendor: 748/32768 files (0.9% non-contiguous), 120659/131072 blocks
```

***

### 1.39.2. 使用Android自带工具badblocks

+ [调试笔记 --- eMMC坏块测试](https://blog.csdn.net/kris_fei/article/details/77316961?utm_source=app&app_version=5.3.1)
+ [Android性能分析之emmc坏块测试](https://blog.csdn.net/Ian22l/article/details/117085760?utm_source=app&app_version=5.3.1)

> android支持emmc坏块测试工具badblocks

**代码路径:**`./external/e2fsprogs/misc/badblocks.c`

默认不编译，需配置：`PRODUCT_PACKAGES += badblocks`

当然你也可以为了开始验证可以直接使用如下命令进行单独编译，前提是你之前已经全部编译通过了。

```shell
source ./build/envsetup.sh
lunch platform_name        ###(platform_name对应的sdk）
mmm external/e2fsprogs/misc/
```

然后将其push到`system/bin`目录下

**使用方法：**
+ [badblocks坏道检测](http://t.zoukankan.com/52why-p-12363039.html)

1.对硬盘进行只读扫描，自动获取硬盘块数目并扫描全部块，将扫描日志输出到屏幕同时记录在sdbbadblocks.log文件中。

由于扫描速度比较低，一次不一定能扫完，可以分多次进行。

+ `sudo badblocks -s -v -o sdbbadblocks.log /dev/sdb END START`
+ `badblocks -sv /dev/block/mmcblk0p10`

2.如果找到了坏道，可以进行写入扫描进行修复。写入扫描遇到坏道的时候会自动重映射。写入扫描会覆盖原有数据，所以请先备份。写入扫描速度很低，所以应该只扫描只读扫描时候发现错误的部分。

`sudo badblocks -w -s /dev/sdb END START`

***




***

## 1.40. Android property属性

+ 参考：[Android 添加系统属性](https://blog.csdn.net/u014674293/article/details/120670723)

在 Android 系统中有一个Property Service服务，这个服务对外提供了两个接口：

+ SystemProperties.get(String key, String def) 读取系统属性
+ SystemProperties.set(String key, String val) 设置系统属性

**特殊前缀属性含义：**
+ ro：只读属性，不能修改。
+ persist：修改属性后，重启依然有效。数据会保存到`/data/property`目录。其他前缀的属性被设置后，只是保存在内存中而已，并没有保存到磁盘，所有重启后就恢复默认值了
+ ctrl：用来启动和停止服务。每一项服务必须在`init.rc`中定义。init一旦收到设置`ctrl.start`属性的请求，属性服务将使用该属性值作为服务名找到该服务，启动该服务。这项服务的启动结果将会放入`init.svc.<服务名>`属性中

***

### 1.40.1. property的相关生成文件和设置

> android通过SystemProperties的set和get方法来控制很多东西，一般上层添加一个控制开关可以使用这个方法，在系统里面存在很多个prop文件。它们分别是`system/build.prop`,`system/etc/prop.default`,`vendor/build.prop`,`vendor/default.prop`。下面分别来说下这几个文件的构成。

+ system/build.prop：这个主要是由device\***(platform)sample\product/system.prop,还有在build目录下添加的ADDITIONAL_BUILD_PROPERTIES
+ system/etc/prop.default：主要是系统添加的PRODUCT_SYSTEM_DEFAULT_PROPERTIES
+ vendor/build.prop（比较重要）：主要是系统添加的PRODUCT_PROPERTY_OVERRIDES，添加在device.mk的这个属性会被编译到这里，但是在9.0的系统，加到这里会无效，获取不到值
+ vendor/default.prop（会被同目录的build.prop相同property覆盖）：主要是系统添加的PRODUCT_DEFAULT_PROPERTY_OVERRIDES

***

## 1.41. ANR日志提取

xxx_app_anr这个⽂件，⼀般记录的进程堆栈信息较之traces⽂件少⼀些。

所以，如果有traces，那我们就基于该⽂件分析定位bug。

+ traces⽂件位于`/data/anr/`⽬录下，默认情况下，每发⽣⼀次anr，都会⽣成traces.txt⽂件，但是后发⽣的会将之前的覆盖掉
+ xxx_app_anr⽂件位于`/data/system/dropbox/`⽬录下，按不同压缩包存放。由Dropbox生成

出现ANR可以使⽤`adb pull`命令将⻋机⾥以上两个⽬录提取出来

***

## 1.42. SOC串口禁止/恢复打印内核日志

通过调整printk的打印级别：

+ 禁止打印：dmesg -n1
+ 恢复打印：dmesg -n8

+ 恢复禁止: echo 0 > /proc/sys/kernel/printk
+ 恢复打印: echo 7 > /proc/sys/kernel/printk

***

## 1.43. dump分区镜像内容（dd命令）

+ 参考：[shell dd命令在bs参数太大的时候出现异常的解决方法](https://blog.csdn.net/coraline1991/article/details/120202643)

通过使用dd命令dump出来实际的分区内容，可以用来同原始image比较

`dd if=/dev/block/by-name/mmcblkp*** of=/sdacar/a.img bs=512`

**dump出来的是raw文件，如果像system/vendor等都是sparse文件，需要先使用`simg2img`转换成raw文件（Android编译环境直接使用）**

块大小bs=2x80x18b表示2 × 80 × 18 × 512 = 1474560字节，也就是一张1440 KiB软盘的确切大小

**另一种方式：**
在当前目录生成一个大小为1G的大文件，内容是0

`dd if=/dev/zero of=/test count=2 bs=512M`

`dd if=/dev/zero of=/test count=1 bs=1G`

**注解：**
+ if 输入文件
+ of 输出文件
+ bs 字节为单位的块大小
+ count 被复制的块数
+ /dev/zero 是一个字符设备，不断的返回0值字节

PS：simg2img是一个能把Android sparse 镜像转为row 镜像的工具，进而可以挂载解压

### 1.43.1. 将vendor_b分区写入到vendor_a分区

`console:/dev/block/by-name # dd if=vendor_b of=vendor_a`

### 1.43.2. simg2img

对比img文件类型：

```s
//车机dd出来的img
Debug_OTA_0401_img$ file system_old.img 
system_old.img: Linux rev 1.0 ext4 filesystem data, UUID=4729639d-b5f2-5cc1-a120-9ac5f788683c (extents) (large files) (huge files)

//系统编译的img
$ file system.img 
system.img: Android sparse image, version: 1.0, Total of 1310720 4096-byte output blocks in 86 input chunks.
```

#### 1.43.2.1. 解压打开image文件

采用挂载分区的方式来打开system.img文件

```s
Debug_OTA_0401_img$ file system_old.img 
system_old.img: Linux rev 1.0 ext4 filesystem data, UUID=4729639d-b5f2-5cc1-a120-9ac5f788683c (needs journal recovery) (extents) (large files) (huge files)
Debug_OTA_0401_img$ mkdir 1
Debug_OTA_0401_img$ sudo mount system_old.img 1/ -o loop
```

#### 1.43.2.2. Tips使用方法

**android本身提供了源代码工具在两者之间转换，源代码位于：**

```s
system/core/libsparse/simg2img.c // 将sparse image转换为raw image;
system/core/libsparse/img2simg.c // 将raw image转换为sparse image;
```

如果完整的进行过一次Android的编译，默认会将simg2img当作主机工具编译出来，放在`out/host/linux-x86/bin/simg2img`处

但默认是不会编译img2simg的，我们可以手工进行编译：


```shell
$ .build/envsetup.sh
$ lunch aosp_hammerhead-userdebug
$ make img2simg_host
```

这样就会编译出`out/host/linux-x86/bin/img2simg`

如果要将system.raw.img转换为system.simg，执行：`img2simg system.raw.img system.simg`

***

## 1.44. Android解析image镜像

### 1.44.1. ota升级包

1.解压ota包获取到payload.bin
2.使用工具`payload_dumper-win64`，将其放入payload_input（windows工具）
3.运行payload_dumper.exe
4.image生成在payload_output

### 1.44.2. dump获取车机的image文件

1.进入adb shell
2.在`/dev/block/by-name/`查看分区对应的`/dev/block/mmcblk0p7`
3.使用`dd if=/dev/block/by-name/mmcblkp*** of=/sdacar/a.img bs=512`命令获取到

### 1.44.3. 解析image文件

1.确认文件属性

```s
//车机dd出来的img
Debug_OTA_0401_img$ file system_old.img 
system_old.img: Linux rev 1.0 ext4 filesystem data, UUID=4729639d-b5f2-5cc1-a120-9ac5f788683c (extents) (large files) (huge files)

//系统编译的img
$ file system.img 
system.img: Android sparse image, version: 1.0, Total of 1310720 4096-byte output blocks in 86 input chunks.
```

2.如果是sparse文件属性，则进行转化：(android编译环境)

`simg2img get.img result.img`

3.采用挂载分区的方式来打开img文件

```s
Debug_OTA_0401_img$ file system_old.img 
system_old.img: Linux rev 1.0 ext4 filesystem data, UUID=4729639d-b5f2-5cc1-a120-9ac5f788683c (needs journal recovery) (extents) (large files) (huge files)
Debug_OTA_0401_img$ mkdir 1
Debug_OTA_0401_img$ sudo mount system_old.img 1/ -o loop
```

***

## 1.45. lpunpack工具

> 参考：[Android super.img镜像解包](https://blog.csdn.net/u012045061/article/details/119383397)

1.编译
```shell
source build/envsetup.sh
make lpunpack
```

2.可以用 lpdump 查看镜像的一些信息，信息里就包括了分区名称
```shell
./lpdump super.img_raw(这里是用 simg2img 转换后的文件)
```

***

## 1.46. 升级包制作打包方法

> 参考: [Android Recovery OTA升级（一）—— make otapackage](http://www.codes51.com/article/detail_153362.html)
> 
> 参考：[Android Update Engine分析（八）升级包制作脚本分析](https://blog.csdn.net/guyongqiangx/article/details/82871409)

`make otapackage`是Android Build系统支持的命令，用来生成Recovery系统能够进行升级的zip包

**或者使用target包制作：**

```shell
# 制作全量升级包
$ ./build/tools/releasetools/ota_from_target_files \
    dist-output/bcm7252ssffdr4-target_files-eng.ygu.zip \
    full-ota.zip

# 制作增量升级包
$./build/tools/releasetools/ota_from_target_files \
    -i dist-output/bcm7252ssffdr4-target_files-eng.ygu.zip \
    dist-output-new/bcm7252ssffdr4-target_files-eng.ygu.zip \
    incremental-ota.zip
```

### 1.46.1. 生成target-files.zip

`make target-files-package`

编译出来的默认路径在:

`out/target/product/{机型名}/obj/PACKAGING/target_files_intermediates/***-target_files-eng.***.zip`

***

## 1.47. 如何禁用OTA更新包生成

在所选用的device中BoardConfig.mk文件，修改或者增加一行`TARGET_SKIP_OTA_PACKAGE := true` 即可在构建时不生成ota更新包

## 1.48. 解析升级包payload.bin工具

+ 参考[payload dumper](https://gist.github.com/ius/42bd02a5df2226633a342ab7a9c60f15)

使用payload dumper工具对升级包patload.bin文件进行解析，可以生成对应升级的image镜像文件

***

## 1.49. android命令打开触摸坐标显示

+ 打开：`settings put system pointer_location 1`
+ 关闭：`settings put system pointer_location 0`

***


***

## 1.50. 解决adb devices报错no permissions(user mi is not in the plugdev group)

`sudo vim /etc/udev/rules.d/android.rules`

`SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", MODE="0666"`

重新拔插一下手机即可

***

## 1.51. adb打开某应用

+ 查看应用包名`dumpsys package`
+ 打开具体包查看activity:`dumpsys package com.***.***`
+ 查看包位置:`adb shell pm path com.***.***`
+ 打开应用：`am start ***/***.actvity`
+ 查看当前打开应用的activity：`dumpsys activity top|grep ACTIVITY`

**例如打开CarSettings应用：**

```shell
/system/priv-app/CarSettings # dumpsys package|grep "car.settings"
        d1e0997 com.android.car.settings/.common.CarSettingActivity (2 filters)
        .....
/system/priv-app/CarSettings # dumpsys activity top|grep ACTIVITY
  ACTIVITY com.android.car.settings/.common.CarSettingActivity 154d0d8 pid=6296

/system/priv-app/CarSettings # am start com.android.car.settings/.common.CarSettingActivity
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.android.car.settings/.common.CarSettingActivity }
Warning: Activity not started, intent has been delivered to currently running top-most instance.
```

***

### 1.51.1. adb命令获取当前打开应用包名

+ windows:`adb shell dumpsys window | findstr mCurrentFocus`
+ linux:`adb shell dumpsys window | grep mCurrentFocus`

***

## 1.52. init.rc文件创建空zip文件

android的init rc目前不支持touch:
+ touch /data/misc/logd/kmsg.log

**log中会报错:init: /init.rc: 83: invalid keyword 'touch'**

**可以用copy和write命令创建文件:**
+ write/data/non-empty-file 
+ copy /dev/null    xxx   //创建xxx文件

***

## 1.53. .rc文件添加服务启动

+ 参考[Android init.rc 添加自定义服务](https://blog.csdn.net/tq501501/article/details/103556837)
+ 参考[LOCAL_PROPRIETARY_MODULE属性](https://deepinout.com/android-mk-explanation/android-mk-module-description-variable/android-mk-local_proprietary_module.html)
+ 参考[SELinux 安全上下文](https://lishiwen4.github.io/android/selinux-security-context)

```s
.rc文件：
//服务定义：
service screencnap/system/bin/test_service
# class 包括core main late_start等，可以不设置。默认default
class main
# 用户属性，默认root，一般给root即可，权限最大
user root
# 所属组，默认root
group root
# disabled 即通过class不能启动，只能start screencap 这种name的方式启动
disabled
# 只运行一次
oneshot

//服务启动条件:
on property:sys.start_service=1
start test_service
```

### 1.53.1. 模块目录将服务名添加到mk/bp文件

```s
//Android.mk
LOCAL_INIT_RC := test.rc

//因为该属性设成true表示vendor专属模块
//如果要移到vendor/bin，则注释掉proprietary : true

//Android.bp
init_rc: ["test.rc"],
```

### 1.53.2. device.mk添加该服务

添加目录：device/project.mk

```s
# test service
PRODUCT_PACKAGES += \
    test_service
```

### 1.53.3. 权限初始添加

添加目录：device/project/sepolicy/vendor/common/

1. 修改device/project/sepolicy/vendor/common/file_contexts
```s
# test service
/vendor/bin/test_service u:object_r:test_service_exec:s0
```

2. 新增device/project/sepolicy/vendor/commo/test_service.te

***


## 1.54. 系统分区RWX权限

> [Android启动过程深入解析](https://blog.csdn.net/jaczen/article/details/72999945)
> [android10.0的init执行顺序](https://blog.csdn.net/weixin_42135087/article/details/104824248)
> [深入理解SELinux SEAndroid（第一部分）](https://blog.csdn.net/innost/article/details/19299937)
> [Android 8.1RK平台增加自定义脚本，修改文件权限](https://blog.csdn.net/weixin_35649059/article/details/106406696)

### 1.54.1. 权限查看修改方法

```s
# ls -al
total 36
drwxrwxrwx  4 system system  4096 2021-12-12 08:00 .
drwxr-xr-x 25 root   root    4096 2022-03-15 18:48 ..
drwx------  2 root   root   16384 1970-01-01 08:00 lost+found
drwx------  3 system system  4096 2021-12-12 08:00 usb
```

`r读 数字4，w写数字2，x执行数字1，3个一组`

`第一组是自己的权限，第二组是自己所在组的权限，第三组是所有人的权限`

+ 修改权限方法:`chmod 764 文件名`，表示`7=4+2+1，6=4+2,4=4`

***

### 1.54.2. 文件夹权限（无法打开读写）

`sudo chmod -R 777 folderName`

***

## 1.55. 查看文件权限和SELinux域

```shell
adb shell
/sdcard/Android/data # ls -lAZ
total 4
-rw-rw---- 1 root   sdcard_rw u:object_r:sdcardfs:s0    0 2021-01-01 08:00 .nomedia
drwxrwx--x 3 system sdcard_rw u:object_r:sdcardfs:s0 4096 2021-01-01 08:00 com.android.media
```

***

## 1.56. selinux对某个文件的权限（有neverAllow）处理方法

`cat log.txt | grep avc | audit2allow`

方法一：（出现naverAllow错误）

> 参考：[Android O selinux违反Neverallow解决办法](https://blog.csdn.net/mafei852213034/article/details/109817800)

```log
type=1400 audit(0.0:124): avc: denied { write } for name="com.engineeringmode-Ok7vxJdZR3gumPFFem-2fg==" dev="mmcblk0p43" ino=8777 scontext=u:r:system_app:s0 tcontext=u:object_r:apk_data_file:s0 tclass=dir permissive=0
```

根据这个mmcblk0p43去在file_context添加：`/dev/block/mmcblk0p43      u:object_r:mmcblk_device:s0`

然后对mmcblk_device赋权限

***

方法二： 

> 参考：[Android6.0 selinux没有对某个文件的权限neverAllow处理方法](https://blog.csdn.net/ly890700/article/details/54645212)

```log
01-01 08:03:22.410000   217   217 W applypatch: type=1400 audit(0.0:16): avc: denied { read } for name="mmcblk0p15" dev="tmpfs" ino=3364 scontext=u:r:install_recovery:s0 tcontext=u:object_r:block_device:s0 tclass=blk_file permissive=0
```

意思是说明install_revovery没有block_device的权限

只要在`install_recovery.te`中加入下面权限就可以了。

`allow install_recovery recover_block_device:blk_file { open read write };`

***

## 1.57. android Selinux 之 platform_app，system_app，priv_app，untrusted_app

> 概念：平台签名：Android.mk中，定义`LOCAL_CERTIFICATE := platform`

> system权限：AndroidManifest.xml中声明`android:sharedUserId="android.uid.system"`，同时是平台签名

**分类：**
+ untrusted_app: 第三方app，没有Android平台签名，没有system权限
+ platform_app: 有android平台签名，没有system权限
+ system_app: 有android平台签名和system权限
+ priv_app 没有platform签名的app(肯定没有system权限)， 但Android.mk 中`LOCAL_PRIVILEGED_MODULE := true`， 在priv-app目录下的app查看

备注：ps只能查看正在运行的进程，如果需要查看指定的app，需要先运行该app

+ 查看全部app类型：`adb shell ps -Z -e`
+ 过滤查看：`adb shell ps -Z -e |grep xxx  或者  adb shell ps -Z -e |findstr xxx`

输出的结果第一列是SContext，第二列是UID，只要UID是system的基本都是system_app

***

## 1.58. selinux报错unlabel

```log
Line 43826: 01-01 09:44:32.220  5076  5076 W engineeringmode: type=1400 audit(0.0:540): avc: denied { search } for name="/" dev="mmcblk0p41" ino=2 scontext=u:r:system_app:s0 tcontext=u:object_r:unlabeled:s0 tclass=dir permissive=0
    Line 43826: 01-01 09:44:32.220  5076  5076 W engineeringmode: type=1400 audit(0.0:540): avc: denied { search } for name="/" dev="mmcblk0p41" ino=2 scontext=u:r:system_app:s0 tcontext=u:object_r:unlabeled:s0 tclass=dir permissive=0
```

假设mmcblk0p41节点映射的是system语音分区，该分区是unlabeled，即系统没有指定的目录。
但是在sepolicy已经对该目录打了标签，所以对该分区路径重新打签名

**重新relable指定的文件(夹)：**

```shell
/dev/block/by-name $ ls -al
...
lrwxrwxrwx 1 root root   21 1970-01-01 08:00 system -> /dev/block/mmcblk0p41
```

```shell
# 修改device/project/init.project.rc
 on fs
    restorecon_recursive /system
```

编译权限生成在系统目录：`/vendor/etc/selinux`

+ 可以使用命令调试：`restorecon -R /（对应权限目录）`

***

## 1.59. 代码关闭selinux权限

> 参考：[android11 关闭 Selinux 的方法](https://blog.csdn.net/lwz622/article/details/122014647)

修改代码`android/system/core/init/selinux.cpp`：`IsEnforcing() 添加：return false`

***

## 1.60. SElinux适配（Android 11&Android 12变更）

> 参考Google官网解释（繁体）https://source.android.google.cn/docs/security/features/selinux/build

Android 12源码说明，BOARD_PLAT_PUBLIC_SEPOLICY_DIR和BOARD_PLAT_PRIVATE_SEPOLICY_DIR被启用，使用SYSTEM_EXT_*替代

```shell
//system/sepolicy/README 
SYSTEM_EXT_PUBLIC_SEPOLICY_DIRS += device/acme/roadrunner-sepolicy/systemext/public
SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS += device/acme/roadrunner-sepolicy/systemext/private
PRODUCT_PUBLIC_SEPOLICY_DIRS += device/acme/roadrunner-sepolicy/product/public
PRODUCT_PRIVATE_SEPOLICY_DIRS += device/acme/roadrunner-sepolicy/product/private

The old BOARD_PLAT_PUBLIC_SEPOLICY_DIR and BOARD_PLAT_PRIVATE_SEPOLICY_DIR
variables have been deprecated in favour of SYSTEM_EXT_*.
```

地点|	包含
:-:|:-:
system/sepolicy/public	|平台的 sepolicy API
system/sepolicy/private	|平台实现细节（产商可以忽略）
system/sepolicy/vendor	|供应商可以使用的策略和上下文文件（如果需要，供应商可以忽略）
BOARD_SEPOLICY_DIRS	|供应商政策
BOARD_ODM_SEPOLICY_DIRS |（Android 9 及更高版本）	odm sepolicy
SYSTEM_EXT_PUBLIC_SEPOLICY_DIRS （Android 11 及更高版本）|	System_ext 的 sepolicy API
SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS （Android 11 及更高版本）|	System_ext 实现细节（产商可以忽略）
PRODUCT_PUBLIC_SEPOLICY_DIRS |（Android 11 及更高版本）	产品的 sepolicy API
PRODUCT_PRIVATE_SEPOLICY_DIRS |（Android 11 及更高版本）

***

## 1.61. git去掉文本末尾的^M符号

执行`git config --global core.whitespace cr-at-eol`

***

## 1.62. Android分区太大编译失败

尝试下面操作：编译失败后
1） 切换到Android根目录，先source lunch，然后执行: `make api-stubs-docs-update-current-api`
2） 再整体编译下

***

# 2. windows技巧

## 2.1. windows获取文件hash哈希值

1. `win+x`键选择PowerShell（增强型命令提示符），或者`win+r`输入`PowerShell`进入终端
2. 执行`get-filehash usb_ota_update.zip -algorithm md5`

**输出：**

```log
get-filehash usb_ota_update.zip -algorithm md5
Algorithm       Hash                                                                   Path
---------       ----                                                                   ----
MD5             5FD6FA7C9C9BCCE6A18102D8A1C4250D                                       D:\...
```

**或者使用certutil命令获取md5值：**

```shell
certutil -hashfile usb_ota_update.zip MD5
结果：
MD5 的 usb_ota_update.zip 哈希:
5fd6fa7c9c9bcce6a18102d8a1c4250d
CertUtil: -hashfile 命令成功完成。
```

**还可以使用：**
+ certutil -hashfile D:\1.exe MD5
+ certutil -hashfile D:\1.exe SHA1
+ certutil -hashfile D:\1.exe SHA256

**对应Ubuntu上就是:**

+ md5sum file
+ sha1sum file
+ sha256sum file

**Tips：**如果出现两个系统对比的文件校验码不同，是因为ftp上传得过程中采用了文本模式，会把文件中换行回车替换为换行。可以用二进制模式上传。（文件打开读取要用二进制方式，文件传输也要用二进制方式）

***

### 2.1.1. md5sum整个文件夹下的文件

`find ./ -type f -print0 | xargs -0 md5sum`

***

## 2.2. Windows强制删除文件及文件夹命令

+ 删除文件或目录CMD命令：
+ + `rd/s/q D:\app`：强制删除文件文件夹和文件夹内所有文件
+ + `del/f/s/q D:\app.txt`：强制删除文件，文件名必须加文件后缀名

+ 删除文件或目录BAT命令：
+ + 新建.BAT批处理文件输入如下命令，然后将要删除的文件拖放到批处理文件图标上即可删除：`DEL /F /A /Q RD /S /Q`

***

***

# 3. linux技巧

## 3.1. 守护进程Daemon

> Linux Daemon（守护进程）是运行在后台的一种特殊进程。它独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。它不需要用户输入就能运行而且提供某种服务，不是对整个系统就是对某个用户程序提供服务。
> 
> 守护进程一般在系统启动时开始运行，除非强行终止，否则直到系统关机都保持运行。守护进程经常以超级用户（root）权限运行，因为它们要使用特殊的端口（1-1024）或访问某些特殊的资源。
> 
> 一个守护进程的父进程是init进程，因为它真正的父进程在fork出子进程后就先于子进程exit退出了，所以它是一个由init继承的孤儿进程。守护进程是非交互式程序，没有控制终端，所以任何输出，无论是向标准输出设备stdout还是标准出错设备stderr的输出都需要特殊处理。
> 
> 守护进程的名称通常以d结尾，比如sshd、xinetd、crond等

***

## 3.2. tree命令使用

```s
tree [-aACdDfFgilnNpqstux][-I <范本样式>][-P <范本样式>][目录...]

-a 显示所有文件和目录
-d 显示目录名称而非内容
-D 列出文件或目录的更改时间。
-f 在每个文件或目录之前，显示完整的相对路径名称
-L level 限制目录显示层级，例如 -L 1

-s 列出文件或目录大小
-t 用文件和目录的更改时间排序
```

***

## 3.3. 重启ubuntu虚拟机后重新挂载

+ 重启命令：`sudo shutdown -r now`
+ 重新挂载：`sudo mount -t ext4 /dev/xvdc /home/user/C`

***

## 3.4. 清理系统

+ `sudo apt-get autoclean`:将已经删除了的软件包的.deb安装文件从硬盘中删除掉
+ `sudo apt-get clean`:删除包缓存中的所有包，也会把你已安装的软件包的安装包也删除掉
+ `sudo apt-get autoremove`:删除为了满足其他软件包的依赖而安装的，但现在不再需要的软件包（[或使用ubuntu-tweak工具清理](https://www.cnblogs.com/boyzgw/p/6610510.html)）
+ `apt-get remove PackageName`: 删除已安装的软件包（保留配置文件）
+ `apt-get --purge remove PackageName`: 删除已安装包（不保留配置文件)

***

## 3.5. 文件权限从root改成system

+ `chown system 文件名`
+ `chgrp system 文件名`

***

## 3.6. beyond compare文件diff工具

> 参考[ubuntu 安装 Beyond Compare 安装，永久破解方法](https://www.jianshu.com/p/93303b9fb21a)

单个文件可以直接使用linux的diff命令

***

## 3.7. 前10个列表

`ls -tl| head -n 10`

## 3.8. 压缩解压命令

+  tar：

同样适用于windows系统，速度快

```shell
# 例子：把/test文件夹打包后生成一个/home/test.tar.gz的文件
tar -zcvf /home/test.tar.gz /test
tar -zcvf 打包后生成的文件名全路径 要打包的目录

# 解压：
tar -xvf test.tar.gz
```

+ zip 压缩方法：

```shell
压缩当前的文件夹 zip -r ./test.zip ./* -r表示递归
zip [参数] [打包后的文件名] [打包的目录路径]
```

+ 解压`unzip test.zip`：如果要把文件解压到指定的目录下,需要用到-d参数，`unzip -d /temp test.zip`

***

## 3.9. GNU nano使用保存退出的说明

文件编辑中常用快捷键：ctrl+X 离开nano软件，若有修改过的文件会提示是否保存；选择 ：yes

又提示：file name to write ：***.launch ，选择：`Ctrl+T`

在下一个界面用 “上下左右” 按键 选择要保存的文件名，然后直接点击 “Enter” 按键即可保存

+ ctrl+O 保存文件； ctrl+W 查询字符串
+ ctrl +C 说明目前光标所在处的行数和列数等信息
+ ctrl+ _ 可以直接输入行号，让光标快速移到该行

***

## 3.10. ubuntu无法使用命令ll

`source .bashrc`

## 3.11. linux查看文件夹大小

```s
system$ du -sh
1.7G	.
system$ du -sh -k
1746700	.
system$ du -sh -m
1706	
```

***

## 3.12. ubuntu磁盘挂载

1.查看新增的磁盘：`sudo fdisk -l`

```shell
.....
Disk /dev/xvdc: 500 GiB, 536870912000 bytes, 1048576000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

2.创建分区

```shell
~$ sudo fdisk /dev/xvdc 

Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x990e1f81.

Command (m for help): n   //输入n，创建新逻辑磁盘
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):     //回车

Using default response p.
Partition number (1-4, default 1): 1   //输入1
First sector (2048-1048575999, default 2048):   //回车
Last sector, +sectors or +size{K,M,G,T,P} (2048-1048575999, default 1048575999):    //回车

Created a new partition 1 of type 'Linux' and of size 500 GiB.

Command (m for help): ^C   //ctrl+c
```

3.磁盘格式化：`sudo mkfs -t ext4 /dev/xvdc`

4.挂载

```shell
~$ mkdir workspace
~$ sudo mount -t ext4 /dev/xvdc /home//workspace
~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            7.9G     0  7.9G   0% /dev
tmpfs           1.6G  9.7M  1.6G   1% /run
/dev/xvda1       83G  5.1G   74G   7% /
tmpfs           7.9G   20M  7.9G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/xvdb1       13M  5.8M  7.3M  45% /mnt/iddisk
tmpfs           1.6G   32K  1.6G   1% /run/user/108
127.0.0.1:/     2.0G 1000M 1000M  50% /ctxmnt
tmpfs           1.6G  120K  1.6G   1% /run/user/16777216
/dev/xvdc       493G   70M  467G   1% /home//workspace
```

**PS：重启虚拟机后，如果硬盘没有自动挂载，重新执行 挂载 这一步**

5.更改用户组

```shell
~$ sudo chown -R : workspace/    //用户冒号后面一个空格然后按tab选择挂载的文件夹
~$ ll
...
drwxr-xr-x  3  domain users 4096 Aug 10 14:35 workspace/  //原来是root
```

6.自动挂载硬盘

```shell
~$ cat /etc/fstab 
.....
# swap was on /dev/xvda5 during installation
UUID=8e68541c-22a5-4156-8bae-c4479adf10d4 none            swap    sw              0       0
/dev/xvdc /home/USER/workspace ext4 defaults 0 1   //新增行在最后
```

***

## 3.13. ubuntu当在终端执行sudo命令时，系统提示“lizh is not in the sudoers file”

**其实就是没有权限进行sudo，解决方法如下：**

1. 切换到超级用户：`su root`
2. 打开`/etc/sudoers`文件：`vim /etc/sudoers`
3. 修改文件内容：找到`root ALL=(ALL) ALL`一行，在下面插入新的一行，内容是`lizh ALL=(ALL) ALL`，然后保存并退出
4. 退出超级用户

***

## 3.14. deb安装包安装命令

安装一个Debian软件包，如你手动下载的文件: `dpkg -i <package.deb>`

```shell
删除软件包 dpkg -r xxx.deb
连同配置文件一起删除 dpkg -r --purge xxx.deb
查看软件包信息 dpkg -info xxx.deb
查看文件拷贝详情 dpkg -L xxx.deb
查看系统中已安装软件包信息 dpkg -l
重新配置软件包 dpkg-reconfigure xxx
```

***

## 3.15. vim设置80字符对齐线

> 参考：[代码规范参考](https://www.jianshu.com/p/45c1675bec69)

`set cc=100` (gerrit上行限制字符数是100)

***

## 3.16. 通过`which +命令`获取该命令的指向路径

## 3.17. linux read()函数和write()函数

+ `ssize_t read(int fildes, void *buf, size_t nbyte);`

```
返回值:
　　> 0: 实际读到的字节数
　　= 0: 读完数据(读文件, 管道, socket末尾-->对端关闭, 对端未关闭会一直等待)
　　-1: 异常:
　　　　errno == EINTR被信号中断, 重启或者退出
　　　　errno == EAGAIN或者EWOULDBLOCK以非阻塞方式读, 并且没有数据
　　　　其他值: 出现错误perror eixt
```
+ `ssize_t write(int fildes, const void *buf, size_t nbyte);`

`返回值: 返回实际写出的字节数, 0表示什么也没有写`

***

## 3.18. ubuntu新建用户

**ubuntu初始环境只有root用户，为了安全，新建一般权限用户。**

1. 添加用户名为test:`root@iZbp10p2g1civut:/# useradd test`
2. 为test用户创建密码，输入该命令后会输入密码，和密码确认：`root@iZbp10p2g1civut:/# passwd test`
3. 为test用户指定命令解释程序（通常为/bin/bash）：`root@iZbp10p2g1civut:/# usermod -s /bin/bash test`
4. 为test用户指定用户主目录：`root@iZbp10p2g1civut:/# usermod -d /home/csdn test`
5. 创建test用户主目录文件夹:`root@iZbp10p2g1civut:/#  mkdir /home/test`
6. 把test文件夹所有权赋给test用户:`root@iZbp10p2g1civut:/#  chown -R test:test /home/test`

此时创建用户已经成功，可以用test用户进行登录。但此时的test用户不能使用sudo命令，因为没有在`/etc/sudoers`文件里给test用户添加权限。

此时切换到root用户下，在`/etc/sudoers`文件中添加一行命令：`test ALL=(ALL) ALL`

**注意：**
1. `test ALL=(ALL) ALL`：允许用户youuser执行sudo命令(需要输入密码)
2. `%test ALL=(ALL) ALL`：允许用户组youuser里面的用户执行sudo命令(需要输入密码)
3. `test ALL=(ALL) NOPASSWD: ALL`：允许用户youuser执行sudo命令,并且在执行的时候不输入密码
4. `%test ALL=(ALL) NOPASSWD: ALL`：允许用户组youuser里面的用户执行sudo命令,并且在执行的时候不输入密码

### 3.18.1. ubuntu查看所有用户

```shell
root@compile03-Virtual-Machine:/etc# grep bash /etc/passwd
root:x:0:0:root:/root:/bin/bash
compile03:x:1000:1000:compile03,,,:/home/compile03:/bin/bash
uchej537:x:999:999::/home/uchej537:/bin/bash
usunw074:x:1001:1001::/home/compile03/workspace/usunw074:/bin/bash
root@compile03-Virtual-Machine:/etc#
```

### 3.18.2. ubuntu切换用户

1. root命令：`sudo su`
2. 切换其他账户：`su 账户名`

### 3.18.3. 为Ubuntu新创建用户创建默认.bashrc并自动加载

1. 拷贝.bashrc:`cp /etc/skel/.bashrc ~/`
2. 拷贝.profile:`cp /etc/skel/.profile ~/`
3. `source ~/.profile`

***

## 3.19. cpu核数

`cat /proc/cpuinfo`

然后查看一共有几核

⼀般⾄少留⼀个核给其他软件使⽤，剩余的拿来编Android。⽽不是上去直接`-j16`，这样很⼤概率不会起到编译加速的作⽤，反⽽会减慢编译速度。

例如9核CPU，⼀般情况下`make -j8`；挂机编译时候直接给满:make -j9`

***

## 3.20. ubuntu/linux查看内存大小

```shell
sudo dmidecode|grep -P -A5 "Memory\s+Device"|grep Size|grep -v Range
[sudo] password for user: 
	Size: 16376 MB

# 或者使用
free -m
```

***

# 4. C/C++编程技巧

## 4.1. new和getInstance()区别

+ **new的使用**:如Object _object = new Object()，这时候，就必须要知道可能会有第二个Object的存在，而第二个Object也常常是在当前的域中的，可以被直接调用的
+ **GetInstance的使用**:在主函数开始时调用，返回一个实例化对象。此对象是static的，在内存中保留着它的引用，即内存中有一块区域专门用来存放静态方法和变量，可以直接使用，调用多次返回同一个对象。

**两者区别对照:**
1. 大部分类(非抽象类/非接口/不屏蔽constructor的类)都可以用new，new就是通过生产一个新的实例对象，或者在栈上声明一个对象 ，每部分的调用用的都是一个新的对象。
2. getInstance是少部分类才有的一个方法，各自的实现也不同。getInstance在单例模式(保证一个类仅有一个实例，并提供一个访问它的全局访问点)的类中常见，用来生成唯一的实例，getInstance往往是static的。

***

**单例模式：一个类有且只有一个实例**
1. 一个私有的构造器
2. 一个私有的该类类型的变量
3. 必须有一个共有的返回类型为该类类型的方法，用来返回这个唯一的变量

***

## 4.2. snprintf()函数用于将格式化的数据写入字符串

其原型为：`int snprintf(char *str, int n, char * format [, argument, ...]);`

+ 【参数】str为要写入的字符串；n为要写入的字符的最大数目，超过n会被截断；format为格式化字符串，与printf()函数相同；argument为变量
+ 【返回值】成功则返回参数str字符串长度，失败则返回-1，错误原因存于errno中

***

## 4.3. 文件操作函数

### 4.3.1. fseek

`int fseek(FILE *stream, long offset, int fromwhere);`：函数设置文件指针stream的位置。

如果执行成功，stream将指向以fromwhere为基准，偏移offset（指针偏移量）个字节的位置，函数返回0。如果执行失败(比如offset取值大于等于2*1024*1024*1024，即long的正数范围2G)，则不改变stream指向的位置，函数返回一个非0值。

fseek函数和lseek函数类似，但lseek返回的是一个off_t数值，而fseek返回的是一个整型。

**具体说明：**

```shell
int fseek( FILE *stream, long offset, int origin );
第一个参数stream为文件指针
第二个参数offset为偏移量，正数表示正向偏移，负数表示负向偏移
第三个参数origin设定从文件的哪里开始偏移,可能取值为：SEEK_CUR、 SEEK_END 或 SEEK_SET
SEEK_SET： 文件开头
SEEK_CUR： 当前位置
SEEK_END： 文件结尾
其中SEEK_SET,SEEK_CUR和SEEK_END依次为0，1和2.
简言之：
fseek(fp,100L,0);把stream指针移动到离文件开头100字节处；
fseek(fp,100L,1);把stream指针移动到离文件当前位置100字节处；
fseek(fp,-100L,2);把stream指针退回到离文件结尾100字节处。
```

### 4.3.2. ftell

`long int ftell(FILE *stream)`：函数 ftell 用于得到文件位置指针当前位置相对于文件首的偏移字节数。在随机方式存取文件时，由于文件位置频繁的前后移动，程序不容易确定文件的当前位置。

`ftell(fp);`:利用函数 ftell() 也能方便地知道一个文件的长。

如以下语句序列： `fseek(fp, 0L,SEEK_END); len =ftell(fp);`

首先将文件的当前位置移到文件的末尾，然后调用函数ftell()获得当前位置相对于文件首的位移，该位移值等于文件所含字节数。

### 4.3.3. feof

feof是C语言标准库函数，其原型在stdio.h中，其功能是检测流上的文件结束符，如果文件结束，则返回非0值，否则返回0（即，文件结束：返回非0值；文件未结束：返回0值）

### 4.3.4. fgets

fgets函数功能为从指定的流中读取数据，每次读取一行。其原型为：`char *fgets(char *str, int n, FILE *stream);`

+ 函数原型:`char *fgets(char *str, int n, FILE *stream);`
+ 参数:
+ + str-- 这是指向一个字符数组的指针，该数组存储了要读取的字符串。
+ + n-- 这是要读取的最大字符数（包括最后的空字符）。通常是使用以 str 传递的数组长度。
+ + stream-- 这是指向 FILE 对象的指针，该 FILE 对象标识了要从中读取字符的流

从指定的流 stream 读取一行，并把它存储在str所指向的字符串内。当读取(n-1)个字符时，或者读取到换行符时，或者到达文件末尾时，它会停止，具体视情况而定

***

## 4.4. ASCII码映射表

![](../../assets/post/2022/2022-12-01-android_debug/ascii码映射表.webp)

### 4.4.1. ASCII转十六进制实现

```c
uint8_t char_2_hex(uint8_t *src)
{
    uint8_t desc;

    if((*src >= '0') && (*src <= '9'))
        desc = *src - 0x30;
    else if((*src >= 'a') && (*src <= 'f'))
        desc = *src - 0x57;
    else if((*src >= 'A') && (*src <= 'F'))
        desc = *src - 0x37;

    return desc;
}
```

其中char字符`buf[1] - 0x30`:表示将16进制数转换成10进制数`0x30在ascii里面是字符0`

### 4.4.2. 十六进制转ASCII实现

```c
uint8_t hex_2_char(uint8_t *src)
{
    uint8_t desc;

    if((*src >= 0) && (*src <= 9))
        desc = *src + 0x30;
    else if((*src >= 0xA) && (*src <= 0xF))
        desc = *src + 0x37;
    
    return desc;
}
```

***

## 4.5. C函数readdir文件夹路径读取

readdir()返回参数dir 目录流的下个目录进入点

d_type表示档案类型：
 
```shell
enum
{
    DT_UNKNOWN = 0,
 # define DT_UNKNOWN DT_UNKNOWN
     DT_FIFO = 1,
 # define DT_FIFO DT_FIFO
     DT_CHR = 2,
 # define DT_CHR DT_CHR
     DT_DIR = 4,
 # define DT_DIR DT_DIR
     DT_BLK = 6,
 # define DT_BLK DT_BLK
     DT_REG = 8,
 # define DT_REG DT_REG
     DT_LNK = 10,
 # define DT_LNK DT_LNK
     DT_SOCK = 12,
 # define DT_SOCK DT_SOCK
     DT_WHT = 14
 # define DT_WHT DT_WHT
};
```

***

## 4.6. C++函数toupper函数或者tolower函数将字符串统一转换为大写或小写然后比较

这种方法不用考虑跨平台的问题，因为使用的是C++标准库中的函数实现的。

```cpp
/*
* 文件名    ：    main.cpp
* 功能        ： 将字符串转换为大写，使用transform函数
*/
#include <iostream>
#include <algorithm>
using namespace std;

int main(int argc, char **argv)
{
    string strTest = "use test.";
    transform(strTest.begin(), strTest.end(), strTest.begin(), toupper);
    for (size_t i = 0; i < strTest.size(); i++) {
        cout << strTest[i];
    }
    cout << endl;
    return 0;
}
```

***

# 5. 工具使用技巧

## 5.1. Android Studio环境配置及常见问题

### 5.1.1. APK缺少签名（system app性质）

**例如：**
1.AndroidManifest.xml中添加：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:sharedUserId="android.uid.system"
    package="com.example.debugapp">
    ....
```

2.build.gradle中添加签名文件

```shell
    signingConfigs {
        debug {
            storeFile file("../signApk/platform.jks")  //导入对应的keystore文件
            storePassword 'android'
            keyAlias 'androiddebugkey'
            keyPassword 'android'
        }
    }
```

**或者这样加：**

```shell
release {
    storeFile file("platform.jks")
    storePassword 'android'
    keyAlias 'platform'
    keyPassword 'android'
    v1SigningEnabled true
    v2SigningEnabled true
}
debug {
    storeFile file("platform.jks")
    storePassword 'android'
    keyAlias 'platform'
    keyPassword 'android'
    v1SigningEnabled true
    v2SigningEnabled true
}
```

3. rebuild

***

#### 5.1.1.1. APP安装报错INSTALL_FAILED_UPDATE_INCOMPATIBLE

```log
adb install -r -t -d Debug.apk
Performing Streamed Install
adb: failed to install Debug.apk: Failure [INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package com.example.debugapp signatures do not match previously installed version; ignoring!]
```

1. 将设备`/data/system/packages.xml`pull到本地
2. 删除该包package中的数据，然后push到设备，重启

***

### 5.1.2. 报错SSL peer shut down incorrectly问题

+ 参考[AndroidStudio SSL peer shut down incorrectly 问题](https://www.jianshu.com/p/194b57cf7162)

**根build.gradle文件中添加镜像仓库:**

```java
buildscript {
    repositories {
        maven { url 'https://jitpack.io' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        google()
        jcenter()
    }
    dependencies {
       .....
    }
}

allprojects {
    repositories {
        maven { url 'https://jitpack.io' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        google()
        jcenter()
    }
``` 

***

### 5.1.3. 报错Caused by: org.gradle.api.internal.plugins.PluginApplicationException: Failed to app

+ 参考[Android Studio出现Caused by: org.gradle.api.internal.plugins.PluginApplicationException: Failed to app](https://blog.csdn.net/weixin_48437363/article/details/109569449)

在项目中的gradle.properties文件中添加语句：

`android.overridePathCheck=true`

***

### 5.1.4. 报错ERROR: Could not install Gradle distribution from ‘https://services.gradle.org/distributions/gradle

网络问题，切换可用网络

***

## 5.2. VSCODE技巧

### 5.2.1. vscode离线插件下载网址

+ 进入[vscode插件官网]（https://marketplace.visualstudio.com/）

### 5.2.2. 思维导图工具

**工具推荐：**
1. [Try markmap在线制作使用markdown](https://markmap.js.org/repl/)
2. [百度脑图在线制作](https://naotu.baidu.com/)：可以导入markdown导出png图片

**vscode插件：**
1. mindmap（文件以km结尾）（参考https://blog.csdn.net/liuxiao723846/article/details/107414365）
2. Markmap（同Try markmap使用markdown，文件以mm.md结尾）

***

### 5.2.3. 代码注释插件

+ 参考：[VScode插件自动添加注释](https://blog.csdn.net/a314753967/article/details/125422370)

安装koroFileHeader插件

**设置选项搜索cursorMode修改如下：**

```shell
 // 函数注释
    "fileheader.cursorMode": {
        // 默认字段
        "description":"",
        "param":"",
        "return":""
```

***

### 5.2.4. vscode添加分隔线

在`file->preferences->settings`的settings.json中添加：`"editor.rulers": [4, 100]`

### 5.2.5. vscode删除缩进多行tab

按：`shift + tab`

***

### 5.2.6. VSCode阅读Android内核源码方法

> 参考：[使用Visual Studio Code阅读Android内核源码](https://www.jianshu.com/p/af723ff252e6)

### 5.2.7. Visual Studio Code（VSCode）关闭右侧预览功能

关闭方法：点击文件-首选项-设置,搜索`editor.minimap.enabled`，默认值为打钩，只需要把钩去掉即可

### 5.2.8. vscode始终打开新标签

`workbench.editor.enablePreview`改为false

### 5.2.9. vscode标签多行显示

`wrap tabs`

### 5.2.10. vscode自动换行

`ALT+Z`

### 5.2.11. markdown pdf转换失败

chromeium安装失败

在首选项-设置-扩展-markdown pdf的executablePath添加路径：`C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe`
