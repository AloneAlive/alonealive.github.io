# types.hal-hardware/interfaces/automotive/vehicle/2.0/types.hal

## 车辆属性VehiclePropertyType
## 车辆区域类型VehicleArea
## 车辆属性分组类型VehiclePropertyGroup
## 车辆窗户区域属性VehicleAreaWindow
## 车辆属性访问权限VehiclePropertyAccess

## 车辆属性变化模式VehiclePropertyChangeMode
### STATIC - 静态属性，不支持subscribe订阅，只能get获取
### ON_CHANGE - 调用get接口获取当前值，set接口是异步的，大多数属性都是这个类型
### CONTINUOUS - 不断的周期性的发送变化。比如速度和方向盘角度等，需要设置采样率范围

## 车辆属性事件订阅信息SubscribeOptions
### int32_t propId - 订阅的属性的Id
### float sampleRate - 标志每秒客户端想要接受多少更新
### SubscribeFlags flags - 指定监听哪个事件的标志

#### UNDEFINED
#### EVENTS_FROM_CAR - 订阅来自vehicle hal的属性，极有可能来自于车辆本身
#### EVENTS_FROM_ANDROID - 当被vehicle hal的client客户端（即car service）调用set时使用该标志订阅事件，并且会触发回调函数onPropertySet

## 车辆座椅属性VehicleAreaSeat

## 车辆属性参数VehiclePropConfig

### int32_t prop - 属性识别码
### VehiclePropertyAccess access - 如果属性可读可写或者均可时定义
### VehiclePropertyChangeMode changeMode 车辆可变属性定义
### vec<VehicleAreaConfig> areaConfigs - 包含每个区域的配置
### vec<int32_t> configArray - 包含额外的配置参数
### string configString - 某些属性可能需要通过此字符串传递的其他信息。 大多数属性不需要设置此项
### float minSampleRate - 以赫兹为单位的最小采样率。必须为VehiclePropertyChangeMode::CONTINUOUS定义
### float maxSampleRate - 以赫兹为单位的最大采样率。必须为VehiclePropertyChangeMode::CONTINUOUS定义

## 接口调用状态值StatusCode

### OK
### TRY_AGAIN
### INVALID_ARG
### NOT_AVAILABLE
### ACCESS_DENIED
### INTERNAL_ERROR