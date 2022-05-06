---
layout: single
related: false
title:  Android Automotive及IVI概述
date:   2021-12-13 14:19:02 +0800
categories: android
tags: IVI android
toc: true
toc_label: "my table"
toc_icon: "cog"
---

Android Automotive及IVI概述

> Android Automotive及IVI概述，对IVI系统，Framework各个模块简单描述

## android Automotive

Android Automotive是⼀个基本的Android平台，它运⾏预安装的（车载信息娱乐）IVI系统，Android应⽤程序以及可选的第⼆⽅和第三⽅Android应⽤程序。

Android Automotive的硬件抽象层(HAL)为Android框架提供了一致的接口（无需考虑物理传输层）。此车载HAL是开发Android Automotive实现的接口。

系统集成商可以将特定于功能的平台HAL接口（如HVAC）与特定于技术的网络接口（如 CAN 总线）连接，以实现车载 HAL 模块。典型的实现可能包括运行专有实时操作系统(RTOS)的专用微控制器单元 (MCU)，该微控制器单元用于CAN总线访问或类似操作，可通过串行链路连接到运行Android Automotive的CPU。

### 和Android平台的关系

+ Android Automotive就是Android平台。Android Automotive并非Android的分支或并行开发版本。它与手机和平板电脑等设备上搭载的 Android 使用相同的代码库，位于同一个存储区中。能够利用现有的安全模型、兼容性计划、开发者工具和基础架构，同时继续保持较高的可定制性和可移植性，完全免费提供并且开源。
+ Android Automotive扩展了Android平台。在将Android打造为功能完善的信息娱乐平台的过程中，增加了对汽车特定要求、功能和技术的支持。Android Automotive将是一个一站式全栈车载信息娱乐平台，就像现在的 Android 系统之于移动设备一样。

### 和Android Auto的区别

+ Android Auto是一个基于用户的手机运行的平台，可通过USB连接将Android Auto用户体验投射到兼容的车载信息娱乐系统。Android Auto支持专为车载用途而设计的应用
+ Android Automotive是直接基于车载硬件运行的操作系统和平台。它是一个可定制程度非常高的全栈开源平台，可为信息娱乐体验提供强大的技术支持。Android Automotive支持专为Android打造的应用，以及专为Android Auto打造的应用

### 架构

车载HAL是汽车与车辆网络服务之间的接口定义（同时保护传入的数据）：

![示意图](/_posts/2021/20211213/vehicle_hal_arch.png)

车载HAL与Android Automotive架构：

+ Car API：内有包含CarSensorManager在内的API。位于/platform/packages/services/Car/car-lib
+ CarService：位于 /platform/packages/services/Car/
+ 车载 HAL：用于定义OEM可以实现的车辆属性的接口。包含属性元数据（例如，车辆属性是否为int以及允许使用哪些更改模式）。位于hardware/libhardware/include/hardware/vehicle.h。如需了解基本参考实现，请参阅 hardware/libhardware/modules/vehicle/（vehicle意思即车辆）

### 系统界面专用组件

组件|说明
:-:|:-:
锁屏界面|用户通过该屏幕向特定用户帐号验证身份。
导航栏|一种系统栏，可以位于屏幕的左侧、底部或右侧，并且可以包含用于导航到不同应用、切换“通知”面板以及提供车辆控制（例如 HVAC）的属性按钮。它与 Android 系统界面实现不同，后者提供返回、主屏幕和应用堆栈按钮。
状态栏|沿屏幕放置的系统栏，用作导航栏。状态栏还提供支持以下各项内容的功能：1.连接图标。包括蓝牙、Wi-Fi 和热点/移动网络连接;2.下拉“通知”面板。例如，从屏幕顶部向下滑动;3.浮动通知 (HUN)
系统界面|指屏幕上显示的任何不属于应用的元素。
用户切换器界面|用户可通过该屏幕选择其他用户。
音量界面|司机使用实体音量按钮改变设备音量时显示的对话框

### 术语名词解释

1. Hvac：供暖通风与空气调节(Heating Ventilation and Air Conditioning)，HVAC 系统可执行多种操作，例如为房屋供暖、为数据中心散热以及控制车载风扇速度
2. Android Automotive：开发汽车应用时所用的嵌入式操作系统和平台
3. Android Auto：Google开发的智能手机投影功能，搭载Android 5.0或更高版本的移动设备可将其应用投影到车载设备上
4. AOSP：全称是Android Open-Source Project，中⽂意思为Android 开放源代码项⽬，⽬前市⾯上基于Android OS的产品基本都是基于AOSP的衍⽣版进⾏⼆次开发（芯⽚公司会基于Google提供的aosp版本进⾏⼆次开发，之后提供给产品公司）
5. GMS：Google Mobile Service，谷歌移动服务。是Google提供的⼀堆服务依赖的系统框架，闭源⽅式提供，如果要使⽤，⼚商需要与Google签授权。国外⼤堆的app依赖GMS，没有GMS，这些app均⽆法正常运⾏
6. GAPPS：Google APPS，由Google提供给厂家的各种app，例如gmail，google maps，google play等，依赖GMS。一般和GMS一样用于国际版，需要Google授权和相关测试认证
7. Google Automotive Services，Google汽车服务 (GAS) 是汽车OEM可以选择授予许可并集成到自己的车载信息娱乐 (IVI) 系统中的应用和服务的集合
8. 汽车测试套件(ATS)：一种测试套件，可验证Android Automotive实现是否按预期运行。例如，ATS测试可能会使用 Car*Manager API来验证车载HVAC集成
9. 板级支持包(BSP)：设备的SoC专用固件
10. **控制器局域网 (CAN)**：一种车载总线标准，允许微控制器与设备相互通信
11. 数字音频广播(DAB)和地面数字音频广播 (T-DAB)：一种音频广播，其中的模拟音频会被转换为数字信号，并通过AM或FM频率范围（更常用）在指定信道上传输
12. 数字版权管理(DRM)：一种系统，通过允许安全分发数据并/或禁止非法分发数据，保护通过互联网或其他数字媒体所传播数据的版权
13. 数字信号处理器 (DSP)：一种专用的微处理器（或SIP块），其架构已经过优化，可满足数字信号处理的各种操作需求。旨在评估、过滤和/或压缩连续的真实环境中的模拟信号
14. 驾驶员分心(DD)	：因某些操作导致驾驶的注意力分散
15. **车载信息娱乐系统(IVI)**：一组可提供音频和/视频娱乐的车载硬件和软件功能。在描述面向用户的Android Automotive设备功能时，通常将该术语作为车机(HU)的同义词
16. 区域互连网路(LIN)：车载组件之间通信时所用的串行网络协议
17. **车载HAL**：该接口会定义原始设备制造商 (OEM)可以实现的属性，并会包含属性元数据（例如，属性是否为int以及允许使用哪些更改模式）
18. 车载地图服务(VMS)：支持高级驾驶辅助系统(ADAS)的车载数据交换服务。允许与其他车载系统共享道路和导航数据，以便众多车载组件和系统在获知道路情况后提供更智能的服务
19. 车辆网络服务(VNS)：通过内置安全机制控制车载HAL。仅限访问系统组件（第三方应用等非系统组件需使用Car API）

### 车辆属性

> 车载硬件抽象层(HAL)接口会定义原始设备制造商(OEM)可以实现的属性，并会包含属性元数据（例如，属性是否为int以及允许使用哪些更改模式）。VHAL接口基于对属性（特定功能的抽象表示）的访问（读取、写入、订阅）

VHAL 使用以下接口：

+ `vehicle_prop_config_t const *(*list_properties)(..., int* num_properties)`:列出 VHAL 所支持的所有属性的配置。车辆网络服务只会使用受支持的属性。
+ `(*get)(..., vehicle_prop_value_t *data)`:读取属性的当前值。对于区域属性，每个区域都可能具有不同的值
+ `(*set)(..., const vehicle_prop_value_t *data)`:为属性写入相应值。写入的结果是按属性进行定义
+ `(*subscribe)(..., int32_t prop, float sample_rate, int32_t zones)`:开始监视属性值的变化。对于区域属性，订阅适用于请求的区域。`Zones=0`用于请求所有受支持的区域。VHAL应该在属性值发生变化时（即变化时触发类型）或按一定间隔（即连续类型）调用单独的回调
+ `(*release_memory_from_get)(struct vehicle_hw_device* device, vehicle_prop_value_t *data)`：释放从get调用分配的内存


VHAL 使用以下回调接口：

+ `(*vehicle_event_callback_fn)(const vehicle_prop_value_t *event_data)`：通知车辆属性值的变化。应只针对已订阅属性执行。
+ `(*vehicle_error_callback_fn)(int32_t error_code, int32_t property, int32_t operation)`:返回全局VHAL级错误或每个属性的错误。全局错误会导致 HAL 重新启动，这可能会导致包括应用在内的其他组件重新启动

#### 属性状态

每个属性值都随附一个VehiclePropertyStatus值。该值指示相应属性的当前状态：

+ AVAILABLE：属性可用，且值有效
+ UNAVAILABLE：属性值目前不可用。该值用于受支持属性的暂时停用的功能
+ ERROR：该属性有问题

***

## IVI系统介绍

> 车载信息娱乐系统（In-Vehicle Infotainment 简称IVI），是采用车载专用中央处理器，基于车身总线系统和互联网服务，形成的车载综合信息处理系统。IVI能够实现包括三维导航、实时路况、IPTV、辅助驾驶、故障检测、车辆信息、车身控制、移动办公、无线通讯、基于在线的娱乐功能及TSP服务等一系列应用，极大的提升了车辆电子化、网络化和智能化水平。

### 术语

1. soc：系统级芯片（system on chip）可将计算机或其他电子系统的所有组件集成到单个芯片的集成电路
2. MCU：微控制单元（Microcontroller Unit），或者叫单片微型计算机，单片机
3. LPDDR4：LPDDR可以说是全球范围内最广泛使用于移动设备的“工作记忆”内存。Low Power Double Data Rate SDRAM，是DDR SDRAM的一种，又称为 mDDR(Mobile DDR SDRM),是美国JEDEC固态技术协会（JEDEC Solid State Technology Association）面向低功耗内存而制定的通信标准，以低功耗和小体积著称，专门用于移动式电子产品。
4. EMMC：eMMC (Embedded Multi Media Card）是MMC协会订立、主要针对手机或平板电脑等产品的内嵌式存储器标准规格。eMMC在封装中集成了一个控制器，提供标准接口并管理闪存，使得手机厂商就能专注于产品开发的其它部分，并缩短向市场推出产品的时间。eMMC 结构由一个嵌入式存储解决方案组成，带有MMC （多媒体卡）接口、快闪存储器设备及主控制器—— 所有在一个小型的BGA 封装。目标应用是对存储容量有较高要求的消费电子产品
5. DSP：数字信号处理（Digital Signal Processing，简称DSP）数字信号处理是利用计算机或专用处理设备，以数字形式对信号进行采集、变换、滤波、估值、增强、压缩、识别等处理，以得到符合人们需要的信号形式
6. audio dsp：音频处理器
7. AMP：电流单位安培，电机的额定电流
8. USART是一个全双工通用同步/异步串行收发模块，该接口是一个高度灵活的串行通信设备
9. HMI：Human Machine Interface，人机接口，也叫人机界面（又称用户界面或使用者界面）
10. VMCU：车载微控制器单元

### IVI硬件划分

硬件按三部分组成：
1. 主机：有三种，分体机，一体机，假一体机类型
2. 屏
3. 外设：包含有线束、DAB广播、天线、Mic麦克风、TBox网络定位、SWC、360倒车摄像头、GPS等

### 接口类型

接口|含义|传输距离
:-:|:-:|:-:
I2C|INTER IC BUS，意为IC之间总线，包括时钟线（SCL）和数据线（SDA）|板级总线
SPI|Serial Peripheral Interface，即串行外设接口|板级总线
UART串口|UART是通用异步收发器（异步串行通信口）的英文缩写，它包括了RS232、RS449、RS423、RS422和RS485等接口标准规范和总线标准规范，即UART是异步串行通信口的总称|RS232是20M，RS485是1KM
CAN总线通信接口|Controller Area Network，即控制器局域网，集成了CAN协议的物理层和数据链路层功能，可完成对通信数据的成帧处理，包括位填充、数据块编码、循环冗余检验、优先级判别等项工作|10KM
usb|usb接口|5M

***

## framework车机模块

Android Framework，可以理解成是Android系统的中间件层，⼤致分为以下⼏部分：
1. Core APP：区别于第三方app和system app，为这两类app提供服务，多为***Provider，源码位于packages/provider和frameworks/base/packages下
2. Framework-java：即除了core app之外的其他java源码部分，例如framework.jar和service.jar
3. Framework-native：c/c++层

### 各模块功能

![IVI各模块功能](211213_androidAutomaticAndIVI_Instruction/framework_IVImoudles.png)

+ Audio：收音机、音频模块，Path:frameworks/av/services/audioflinger/，Android管理来自Android应用的声音，同时控制这些应用，并根据其声音类型将声音路由到HAL中的输出设备

![audio](211213_androidAutomaticAndIVI_Instruction/aaos_audio_01.png)

+ Carservice：car service，Path:packages/services/Car/（后续该模块详细学习）
+ + 按键输入：`packages/services/Car/service/src/com/android/car/CarInputService.java`，Android Automotive根据`hardware/libhardware/include/hardware/vehicle.h`中定义的车载HAL属性VEHICLE_PROPERTY_HW_KEY_INPUT处理来自转向远程开关、硬件按钮和触摸面板等元素的按键输入。例如通过CAN总线网络调度按键事件：

![key input](211213_androidAutomaticAndIVI_Instruction/key_input_01.png)

+ BT&Telecom：蓝牙电话模块，CarBluetoothService维护当前用户的蓝牙设备以及连接到IVI的每个配置文件的优先级列表。设备按指定的优先级顺序连接到配置文件，`Path：packages/services/Car/service/src/com/android/car/CarBluetoothService.java`；CarBluetoothManager会提供connectDevices() API调用，根据为每个蓝牙配置文件定义的优先级列表继续连接设备，`Path：packages/services/Car/car-lib/src/android/car/CarBluetoothManager.java`
+ Media：视频媒体模块，在`frameworks/av/services/`目录下
+ CameraService：camera模块，倒车（有IVC和360两种方式）

+ + EVS应用：可作为参考实现的C++ EVS示例应用 (/packages/services/Car/evs/app)。该应用负责从EVS管理器请求视频帧，并将用于显示的已完成的帧发送回EVS管理器。EVS和汽车服务可供使用后，它便立即由init启动（设置目标为在开机两 (2) 秒内启动）。原始设备制造商(OEM)可视需要修改或替换EVS应用。

![EVS](211213_androidAutomaticAndIVI_Instruction/vhal_evs_components.png)

+ + EVS管理器：(/packages/services/Car/evs/manager)可提供EVS应用所需的构建块，以实现从简单的后视摄像头显示到6DOF多摄像头渲染的任何功能。它的接口通过 HIDL呈现，并且能够接受多个并发客户端。其他应用和服务（特别是汽车服务）可以查询EVS管理器状态，以了解EVS系统何时处于活动状态
+ + EVS HIDL接口：在EVS系统中，相机和显示元素均由android.hardware.automotive.evs 软件包定义。用于实践接口的示例实现（生成合成测试图像并验证图像进行往返的过程）在`/hardware/interfaces/automotive/evs/1.0/default`中提供

+ EOLManager：EOL下线配置
+ TunerManager
+ TBoxManager：Tbox，上网定位
+ UpdateService：升级（包含很多方面，SOC、MCU、屏幕等）
+ Other

### 仪表板

> Instrument Cluster API（仪表组API，一款Android API）可在车载辅助显示设备（如位于方向盘后方的仪表盘上的辅助显示设备）上显示导航应用，包括Google地图。以及创建服务以控制该辅助显示设备并将该服务与CarService集成，以便导航应用可以显示界面

术语|说明
:-:|:-:
CarInstrumentClusterManager|一个CarManager，使外部应用能够在仪表板上启动Activity，并在仪表板准备好显示Activity时接收回调。Path:`packages/services/Car/car-lib/src/android/car/cluster/CarInstrumentClusterManager.java`
CarManager|所有管理器的基类，外部应用使用这些管理器与通过CarService实现的汽车特有服务进行交互
CarService|一种Android平台服务，可在Google地图等外部应用与仪表板等汽车特有功能之间提供通信服务
车机HU|车内嵌入的主要计算单元。HU会运行所有Android代码，并连接到汽车中央显示屏。能够搭载Android 9（或更高版本）的Android设备。此设备必须具有自己的显示屏，并且能够使用Android的新build刷写显示屏
仪表板|位于方向盘后方车载仪表之间的辅助显示设备。这可以是通过汽车内部网络（CAN 总线）连接到HU的独立计算单元，也可以是连接到HU的辅助显示设备
InstrumentClusterRenderingService|用于与仪表板显示屏连接的服务的基类。原始设备制造商(OEM)必须提供该类的扩展，以便与OEM特有硬件互动。Path：`packages/services/Car/car-lib/src/android/car/cluster/renderer/InstrumentClusterRenderingService.java`
KitchenSink应用|Android Automotive中包含的测试应用

1. CarService

CarService可在导航应用与汽车之间进行协调，确保在任何时候只有一个导航应用处于活动状态，并且只有具有 android.car.permission.CAR_INSTRUMENT_CLUSTER_CONTROL 权限的应用才能向汽车发送数据。

CarService可以启动所有汽车特有服务，并通过一系列管理器提供对这些服务的访问。为了与服务进行互动，在汽车内运行的应用可以访问这些管理器。

对于仪表板实现，汽车OEM必须创建自定义的InstrumentClusterRendererService实现，并更新config.xml文件以指向该自定义实现。

当呈现仪表板时，CarService会在启动过程中读取config.xml的InstrumentClusterRendererService密钥，以定位InstrumentClusterService实现。

2. 仪表板服务：OEM必须创建包含InstrumentClusterRendererService子类的Android软件包 (APK)


## 参考

> [spi与i2c区别](http://www.elecfans.com/emb/app/20171109576955.html)
> 
> [I2C、UART串口、SPI、CAN、USB通信接口](https://www.jianshu.com/p/d75c35a92f76)
> 
> [SPI、I2C、UART、CAN](https://blog.csdn.net/u010183728/article/details/81984433)
> 
> [Google开发官网——Automotive](https://source.android.google.cn/devices/automotive?hl=zh-cn)
> 
> [Google开发官网——仪表板](https://source.android.google.cn/devices/automotive/displays/cluster_api?hl=zh-cn)
> 
> [Google开发官网——系统界面](https://source.android.google.cn/devices/automotive/hmi/system_ui?hl=zh-cn)
> 
> [Google开发官网——车辆属性](https://source.android.google.cn/devices/automotive/properties?hl=zh-cn)
