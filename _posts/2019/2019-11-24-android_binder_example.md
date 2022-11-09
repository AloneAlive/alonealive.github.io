---
title: Android Binder实例
toc: true
date: 2019-11-24 15:59:40
tags:
- graphics
categories:
- android
---

> Binder通信是Android用的比较多的一种通信机制，它是一种client-server的通信结构。Binder通信具有以下特点：
1. 用驱动程序来推进进程间的通信；
2. 可以通过共享内存的方式来提供性能；
3. 可以为进程请求分配每个进程的线程池；
4. 针对系统中的对象引入了引用计数和跨进程的对象引用映射；
5. 可以方便的进行进程同步调用。

***

> 以下简单的Binder实例参考一位大佬。

# 1. 文件目录

```shell
 cmds/helloWorld/Android.mk                 | 29 +++++++++++++++++++++++
 cmds/helloWorld/BnHelloWorldService.h      | 16 +++++++++++++
 cmds/helloWorld/BpHelloWorldService.h      | 12 ++++++++++
 cmds/helloWorld/HelloWorldService.h        | 17 +++++++++++++
 cmds/helloWorld/IHelloWorldService.h       | 21 +++++++++++++++++
 cmds/helloWorld/main_helloworldclient.cpp  | 36 ++++++++++++++++++++++++++++
 cmds/helloWorld/main_helloworldservice.cpp | 22 +++++++++++++++++
 libs/helloWorld/Android.bp                 | 38 ++++++++++++++++++++++++++++++
 libs/helloWorld/BnHelloWorldService.cpp    | 24 +++++++++++++++++++
 libs/helloWorld/BnHelloWorldService.h      | 16 +++++++++++++
 libs/helloWorld/BpHelloWorldService.cpp    | 25 ++++++++++++++++++++
 libs/helloWorld/BpHelloWorldService.h      | 12 ++++++++++
 libs/helloWorld/HelloWorldService.cpp      | 33 ++++++++++++++++++++++++++
 libs/helloWorld/HelloWorldService.h        | 17 +++++++++++++
 libs/helloWorld/IHelloWorldService.cpp     |  8 +++++++
 libs/helloWorld/IHelloWorldService.h       | 21 +++++++++++++++++
```

# 2. `cmds/helloWorld/Android.mk`

```makefile
# Copyright 2019 The Android Open Source Project
#
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)
LOCAL_SRC_FILES := main_helloworldservice.cpp

LOCAL_SHARED_LIBRARIES :=\
	libutils \
	libbinder \
	libhelloworld

base := $(LOCAL_PATH)/../../libs/helloWorld

LOCAL_MODULE := helloworldservice
include $(BUILD_EXECUTABLE)

include $(CLEAR_VARS)
LOCAL_SRC_FILES := main_helloworldclient.cpp

LOCAL_SHARED_LIBRARIES :=\
	libutils \
	libbinder \
	libhelloworld

base := $(LOCAL_PATH)/../../libs/helloWorld

LOCAL_MODULE := helloworldclient  //编译结果so文件
include $(BUILD_EXECUTABLE)
```

***

# 3. `cmds/helloWorld/BpHelloWorldService.h`

> 客户端Service头文件，声明`BpHelloWorldService`函数

```cpp
#include <binder/Parcel.h>
#include <IHelloWorldService.h>

namespace android{
class BpHelloWorldService: public BpInterface<IHelloWorldService>
{
public:
    BpHelloWorldService (const sp<IBinder>& impl);
    virtual status_t helloWorld(const char *str);
};
};
```

***

# 4. `cmds/helloWorld/BnHelloWorldService.h`

> Bn服务端Service头文件，声明`onTranscat`接口

```cpp
#ifndef ANDROID_BNHELLOWORLD_H
#define ANDROID_BNHELLOWORLD_H

#include <binder/Parcel.h>
#include <IHelloWorldService.h>

namespace android {
class BnHelloWorldService : public BnInterface<IHelloWorldService>
{
    public:
    virtual status_t onTransact ( uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0 );
};
};
#endif
```

***

# 5. `cmds/helloWorld/HelloWorldService.h`

> Bn服务端实现类的头文件，声明`helloworld`和`onTransact`函数，和私有类构造函数和析构函数  
> 和另一个库的文件同名，继承Bn服务端接口

```cpp
#include <BnHelloWorldService.h>
#include <utils/Log.h>

namespace android {
class HelloWorldService : public BnHelloWorldService
{
public:
        static void instantiate();
        virtual status_t helloWorld(const char *str);
        virtual status_t onTransact(
              uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags);
private:
    HelloWorldService();
    virtual ~HelloWorldService();
};
};
```

***

# 6. `cmds/helloWorld/IHelloWorldService.h`

> Bp和Bn端的中间接口头文件

```cpp
#ifndef ANDROID_HELLOWORLD_H
#define ANDROID_HELLOWORLD_H

#include <binder/IInterface.h>

namespace android {

enum {
    HW_HELLOWORLD = IBinder::FIRST_CALL_TRANSACTION,
};

class IHelloWorldService: public IInterface { 

    public:
    DECLARE_META_INTERFACE(HelloWorldService);
    virtual status_t helloWorld(const char *str) = 0;
};
};

#endif
```

***

# 7. `cmds/helloWorld/main_helloworldclient.cpp`

```cpp
#define LOG_TAG "main_helloworldclient"

#include <binder/IPCThreadState.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>

#include <utils/Log.h>
#include <utils/RefBase.h>
#include <IHelloWorldService.h>

using namespace android;
#define unused(x) x=x

int main(int argc, char *argv[])
{
//ALOGI("HelloWorldService client is now starting");
unused(argc);
unused(argv);

sp<IServiceManager> sm = defaultServiceManager();
sp<IBinder> b;
sp<IHelloWorldService> sHelloWorldService;

do {
    b = sm->getService(String16(
"android.apps.IHelloWorldService"));
    if (b!=0)
    break;
    //ALOGI("helloworldservice is not working, waiting ...");
    usleep(500000);
} while(true);

sHelloWorldService = interface_cast<IHelloWorldService>(b);
sHelloWorldService -> helloWorld("hello, world");

return(0);
}
```

***

# 8. `cmds/helloWorld/main_helloworldservice.cpp`

```cpp
#include <binder/IPCThreadState.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>
#include <utils/Log.h>

#include <IHelloWorldService.h>
#include <HelloWorldService.h>

using namespace android;
#define unused(x) x=x
int main(int argc, char *argv[])
{
   unused(argc);
   unused(argv);
    HelloWorldService::instantiate();
    ProcessState::self()->startThreadPool();
    //ALOGI("HelloWorldService is starting now");
    //ALOGI("HelloWorldService is starting now   tempChar = %s", tempChar);
    IPCThreadState::self()->joinThreadPool();
    return 0;
}
```

***

# 9. `libs/helloWorld/Android.bp`

```makefile
// Copyright (C) 2010 The Android Open Source Project
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
cc_library_shared {
    name: "libhelloworld",
    srcs: [
        "BnHelloWorldService.cpp",
        "BpHelloWorldService.cpp",
        "HelloWorldService.cpp",
        "IHelloWorldService.cpp",
    ],

    shared_libs: [
        "libcutils",
        "liblog",
        "libutils",
        "libbinder",
    ],

    include_dirs: ["frameworks/base/cmds"], //上面创建的库

    cflags: [
        "-Wall",
        "-Wextra",
        "-Werror",
    ],

}
```

***

# 10. `libs/helloWorld/BnHelloWorldService.h`

```cpp
#ifndef ANDROID_BNHELLOWORLD_H
#define ANDROID_BNHELLOWORLD_H

#include <binder/Parcel.h>
#include <IHelloWorldService.h>

namespace android {
class BnHelloWorldService : public BnInterface<IHelloWorldService>
{
    public:
    virtual status_t onTransact ( uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0 );

};
};
#endif
```

***

# 11. `libs/helloWorld/BnHelloWorldService.cpp`

```cpp
#include <BnHelloWorldService.h>
#include <binder/Parcel.h>

namespace android {

status_t BnHelloWorldService::onTransact(uint32_t code,
                              const Parcel &data,
                              Parcel *reply,
                              uint32_t flags)
{
    switch(code) {
    case HW_HELLOWORLD: {
        CHECK_INTERFACE(IHelloWorldService, data, reply);   //检查接口
        const char *str;
        str = data.readCString(); //读取数据
        reply-> writeInt32(helloWorld(str)); //写入数据
        return NO_ERROR;
    } break;
    default:
       return BBinder::onTransact(code, data, reply, flags); //服务端接口接收数据
    }

}
}
```

***

# 12. `libs/helloWorld/BpHelloWorldService.h`

```cpp
#include <binder/Parcel.h>
#include <IHelloWorldService.h>

namespace android{
class BpHelloWorldService: public BpInterface<IHelloWorldService>
{
public:
    BpHelloWorldService (const sp<IBinder>& impl);
    virtual status_t helloWorld(const char *str);
};

};
```

***

# 13. `libs/helloWorld/BpHelloWorldService.cpp`

```cpp
#include <binder/Parcel.h>
#include <BpHelloWorldService.h>
#include <utils/Log.h>

namespace android{

status_t BpHelloWorldService::helloWorld(const char *str) {
    Parcel data, reply;
    data.writeInterfaceToken(
	IHelloWorldService::getInterfaceDescriptor());
    data.writeCString(str);  //写入数据
    status_t status = remote()->transact(HW_HELLOWORLD, data, &reply); //远程传输数据
    if (status != NO_ERROR) {
        ALOGI("print helloworld error : %s", strerror(-status));
    } else {
        status = reply.readInt32();  //读取数据
    }
    return status;
}

BpHelloWorldService::BpHelloWorldService (const sp<IBinder>& impl)
	: BpInterface<IHelloWorldService>(impl)
{}

};
```

***

# 14. `libs/helloWorld/HelloWorldService.h`

```cpp
#include <BnHelloWorldService.h>
#include <utils/Log.h>

namespace android {
//继承Bn服务端
class HelloWorldService : public BnHelloWorldService
{
public:
        static void instantiate();
        virtual status_t helloWorld(const char *str);
        virtual status_t onTransact(
              uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags);
private:
    HelloWorldService();
    virtual ~HelloWorldService();
};
};
```

***

# 15. `libs/helloWorld/HelloWorldService.cpp`

```cpp
#include <binder/IServiceManager.h>
#include <binder/IPCThreadState.h>
#include <BnHelloWorldService.h>
#include <HelloWorldService.h>
#include <utils/Log.h>

namespace android {
void HelloWorldService::instantiate() {
    defaultServiceManager()->addService(
            String16("android.apps.IHelloWorldService"), new HelloWorldService());
}

status_t HelloWorldService::helloWorld(const char* str) {
    ALOGI("%s\n", str);
    printf("%s\n", str);
    return NO_ERROR;
}

HelloWorldService::HelloWorldService(){
    ALOGI("HelloWorldService is created");
}

HelloWorldService::~HelloWorldService(){
    ALOGI("HelloWorldService is destroyed");
}

status_t HelloWorldService::onTransact(
              uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    return BnHelloWorldService::onTransact(code, data, reply, flags);
}

};
```

***

# 16. `libs/helloWorld/IHelloWorldService.h`

```cpp
#ifndef ANDROID_HELLOWORLD_H
#define ANDROID_HELLOWORLD_H

#include <binder/IInterface.h>

namespace android {

enum {
    HW_HELLOWORLD = IBinder::FIRST_CALL_TRANSACTION,
};

class IHelloWorldService: public IInterface { 

    public:
    DECLARE_META_INTERFACE(HelloWorldService);
    virtual status_t helloWorld(const char *str) = 0;
};
};

#endif
```

***

# 17. `libs/helloWorld/IHelloWorldService.cpp`

```cpp
#include <IHelloWorldService.h>
#include <BpHelloWorldService.h>

namespace android {

IMPLEMENT_META_INTERFACE(HelloWorldService, "android.apps.IHelloWorldService");

};
```
