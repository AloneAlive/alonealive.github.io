# Car Service(packages/services/Car/service/)

## CarService

## CarPropertyServiceBase(实现了IcarProperty.aidl接口，控制各manager对车机属性的注册，监听以及读写。当属性发生变化时，回调给注册监听了该属性的manager，然后继续向上传递，使得各属性manager方便的处理车机相关属性。该类对车机属性的处理是通过PropertyHalServiceBase进行的，而在PropertyHalServiceBase中的操作为VehicleHal)

## PropertyHalServiceBase(汽车HAL用来传输车辆属性的服务,为CarPropertyService提供对各属性的操作继承自HalServiceBase，继承HalServiceBase的还有InputHalService,SensorHalService,PowerHalService以及VmsHalService)

## CarNightService(检测白天黑夜状态) - packages/services/Car/service/src/com/android/car/CarNightService.java

## AppFocusService(负责应用焦点抢占管理，确保一次只有一个导航类或语音识别类的实例是活动的)

## CarInputService

## GarageModeService(车库模式服务，当汽车不在使用时，为保养维护设计的一个时间窗口)

## SystemStateControllerService(监听PowerManager的状态，来解除静音等)

## PerUserCarServiceHelper & PerUserCarService & CarBluetoothUserService(因为CarService运行在系统用户上。此服务运行在每个用户交换机上的当前用户，CarService的元件可以使用此服务与当前（非系统）用户交流。如用户切换时，会去替换蓝牙通讯录等)

## ......