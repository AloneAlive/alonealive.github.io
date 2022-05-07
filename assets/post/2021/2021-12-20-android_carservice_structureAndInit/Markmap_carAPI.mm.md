# Car API(同Car Service通过AIDL通信) - packages/services/Car/car-lib/
## android car(包含了与车相关的基本API)
## annotation(包含两个注解)
## app
### menu(车辆应用菜单相关API)
## cluster(仪表盘相关API)
### CarInstrumentClusterManager
### Render(渲染相关API)
## content
### CarPackageManager(监测应用程序安装和刷新策略,允许应用对系统范围内的package进行控制，允许添加黑白名单以便允许或组织APP的运行，例如在高速状态下禁止video应用等)
## diagnostic(包含与汽车诊断相关的API)
### CarDiagnosticManager(支持收集诊断信息从及检测诊断数据等)
## hardware
### CarHvacManager(监测和控制空调状态) - packages/services/Car/car-lib/src/android/car/hardware/hvac/CarHvacManager.java
### CarPowerManager
### CarSensorManager(监测当前车辆行驶状态、白天黑夜、点火状态、倒车状态、总线车速/转速等) - packages/services/Car/car-lib/src/android/car/hardware/CarSensorManager.java
### CarCabinManager(用于检测车舱状态，如座椅、车窗位置等)
### CarPropertyManager(Android P里上面大部分service都用这个代替了)
### CarVendorExtensionManager(用于获取车辆的一些可变属性，自定义car event可以放到其中进行管理,负责处理由原始设备制造商在车辆HAL中定义的自定义属性的服务)
### CarRadioManager&RadioManager(支持预设电台等收音机相关的功能特性)
## CarAppFocusManager(允许应用获取系统当前的焦点应用，例如导航或者语音是否处于active状态)
## CarInfoManager(获取车辆静态信息，例如制造商，VehicleID，生产日期等)
## CarProjectionManager(允许实现投影的应用程序用投影管理器注册/取消注册，监听语音通知)
## CarBluetoothManager(设置特定于汽车的蓝牙连接管理策略的API)
## input(输入相关API)
### CarInputHandlingService(监测语音助手、仪表、音量、拨号接听、上次通话等按键事件)
## media(多媒体相关API)
### CarAudioManager(负责声音抢占管理、音量管理等)
## navigation(导航相关API)
### CarNavigationStatusManager
## settings(设置相关API)
### CarConfigurationManager
## vms(汽车检测相关API)
### VmsSubscriberManager