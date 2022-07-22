---
layout: single
title:  Android AB升级（三） - update engine架构概述
date:   2022-07-20 14:21:02 +0800 
categories: ota 
tags: android ota AB升级
toc: true
---

> update engine是通过AIDL对上层client端和server端分离，实现跨进程。上层提供服务绑定接口，以及升级触发和回调接口，底层实现升级的具体逻辑。本篇只是简单梳理下流程流转的过程。


# 1. 应用升级接口相关文件

1.framework层应用接口（systemApi）
+ frameworks/base/core/java/android/os/UpdateEngineCallback.java
+ frameworks/base/core/java/android/os/UpdateEngine.java

2.AIDL接口文件：
+ system/update_engine/binder_bindings/android/os/IUpdateEngine.aidl
+ system/update_engine/binder_bindings/android/os/IUpdateEngineCallback.aidl

3.服务端接口文件：
+ 继承BnUpdateEngine： system/update_engine/binder_service_android.h
+ 继承BnUpdateEngineCallback： system/update_engine/update_engine_client_android.cc

***

## 1.1. UpdateEngine类接口

UpdateEngine类（@SystemApi）主要提供bind和applyPayload接口给应用

### 1.1.1. 代码流程（bind和applyPayload）

1.UpdateEngine.java

- bind(final UpdateEngineCallback callback, final Handler handler) 主要接受UpdateEngineCallback对象，同步代码块中会实现callback的两个接口，获取升级服务的状态码和结果错误码
- applyPayload传递升级包路径大小等信息，并会传递到服务端进行实际逻辑的操作

```java
//frameworks/base/core/java/android/os/UpdateEngine.java
    //传递升级包信息
    public void applyPayload(String url, long offset, long size, String[] headerKeyValuePairs) {
        try {
            mUpdateEngine.applyPayload(url, offset, size, headerKeyValuePairs);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    public boolean bind(final UpdateEngineCallback callback, final Handler handler) {
        synchronized (mUpdateEngineCallbackLock) {
            //IUpdateEngineCallback对象
            mUpdateEngineCallback = new IUpdateEngineCallback.Stub() {
                //重写回调类的接口，其中会调用到服务端的对应实现接口
                @Override
                public void onStatusUpdate(final int status, final float percent) {
                    if (handler != null) {
                        handler.post(new Runnable() {
                            @Override
                            public void run() {
                                callback.onStatusUpdate(status, percent);
                            }
                        });
                    } else {
                        callback.onStatusUpdate(status, percent);
                    }
                }

                @Override
                public void onPayloadApplicationComplete(final int errorCode) {
                    if (handler != null) {
                        handler.post(new Runnable() {
                            @Override
                            public void run() {
                                callback.onPayloadApplicationComplete(errorCode);
                            }
                        });
                    } else {
                        callback.onPayloadApplicationComplete(errorCode);
                    }
                }
            };

            try {
                return mUpdateEngine.bind(mUpdateEngineCallback);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
    }
```

2.AIDL接口

```cpp
interface IUpdateEngine {
  /** @hide */
  void applyPayload(String url,
                    in long payload_offset,
                    in long payload_size,
                    in String[] headerKeyValuePairs);
  /** @hide */
  boolean bind(IUpdateEngineCallback callback);
  ....
```

3.服务端接口

```cpp
//system/update_engine/binder_service_android.h
class BinderUpdateEngineAndroidService : public android::os::BnUpdateEngine,
                                         public ServiceObserverInterface {
                                             ...

//system/update_engine/binder_service_android.cc
Status BinderUpdateEngineAndroidService::applyPayload(
    const android::String16& url,
    int64_t payload_offset,
    int64_t payload_size,
    const std::vector<android::String16>& header_kv_pairs) {
  const std::string payload_url{android::String8{url}.string()};
  std::vector<std::string> str_headers;
  str_headers.reserve(header_kv_pairs.size());
  for (const auto& header : header_kv_pairs) {
    str_headers.emplace_back(android::String8{header}.string());
  }

  brillo::ErrorPtr error;
  //调用system/update_engine/update_attempter_android.cc的接口
  if (!service_delegate_->ApplyPayload(
          payload_url, payload_offset, payload_size, str_headers, &error)) {
    return ErrorPtrToStatus(error);
  }
  return Status::ok();
}

Status BinderUpdateEngineAndroidService::bind(
    const android::sp<IUpdateEngineCallback>& callback, bool* return_value) {
  callbacks_.emplace_back(callback);

  const android::sp<IBinder>& callback_binder =
      IUpdateEngineCallback::asBinder(callback);
  auto binder_wrapper = android::BinderWrapper::Get();
  binder_wrapper->RegisterForDeathNotifications(
      callback_binder,
      base::Bind(
          base::IgnoreResult(&BinderUpdateEngineAndroidService::UnbindCallback),
          base::Unretained(this),
          base::Unretained(callback_binder.get())));

  // Send an status update on connection (except when no update sent so far),
  // since the status update is oneway and we don't need to wait for the
  // response.
  if (last_status_ != -1)
    callback->onStatusUpdate(last_status_, last_progress_);

  *return_value = true;
  return Status::ok();
}
```

***

## 1.2. UpdateEngineCallback类接口

主要是onStatusUpdate和onPayloadApplicationComplete接口

### 1.2.1. 代码流程（onStatusUpdate和onPayloadApplicationComplete）

1.抽象类，实现在UpdateEngine的bind方法中：
```java
//frameworks/base/core/java/android/os/UpdateEngineCallback.java
@SystemApi
public abstract class UpdateEngineCallback {
    public abstract void onStatusUpdate(int status, float percent);
    public abstract void onPayloadApplicationComplete(int errorCode);
}
```

2.AIDL接口：

```java
//system/update_engine/binder_bindings/android/os/IUpdateEngineCallback.aidl
oneway interface IUpdateEngineCallback {
  /** @hide */
  void onStatusUpdate(int status_code, float percentage);
  /** @hide */
  void onPayloadApplicationComplete(int error_code);
}
```

3.服务端接口：

```cpp
//system/update_engine/update_engine_client_android.cc
private:
  class UECallback : public android::os::BnUpdateEngineCallback {
   public:
    explicit UECallback(UpdateEngineClientAndroid* client) : client_(client) {}

    // android::os::BnUpdateEngineCallback overrides.
    Status onStatusUpdate(int status_code, float progress) override;
    Status onPayloadApplicationComplete(int error_code) override;

   private:
    UpdateEngineClientAndroid* client_;
  };
...
  android::sp<android::os::IUpdateEngine> service_;
  android::sp<android::os::BnUpdateEngineCallback> callback_;
};
//升级过程中会打印这两个接口的日志
Status UpdateEngineClientAndroid::UECallback::onStatusUpdate(int status_code,
                                                             float progress) {
  update_engine::UpdateStatus status =
      static_cast<update_engine::UpdateStatus>(status_code);
  LOG(INFO) << "onStatusUpdate(" << UpdateStatusToString(status) << " ("
            << status_code << "), " << progress << ")";
  return Status::ok();
}

Status UpdateEngineClientAndroid::UECallback::onPayloadApplicationComplete(
    int error_code) {
  ErrorCode code = static_cast<ErrorCode>(error_code);
  LOG(INFO) << "onPayloadApplicationComplete(" << utils::ErrorCodeToString(code)
            << " (" << error_code << "))";
  client_->ExitWhenIdle(
      (code == ErrorCode::kSuccess || code == ErrorCode::kUpdatedButNotActive)
          ? EX_OK
          : 1);
  return Status::ok();
}
```

***

# 2. 解析升级包payload.bin工具

+ [payload dumper](https://gist.github.com/ius/42bd02a5df2226633a342ab7a9c60f15)

使用payload dumper对升级包patload.bin文件进行解析，可以生成对应升级的image镜像文件

# 3. 升级系列文章参考

+ *[Android A/B System OTA分析（一）概览](https://blog.csdn.net/guyongqiangx/article/details/71334889)
+ [Android OTA升级原理和流程分析（零）---启动篇](https://blog.csdn.net/twk121109281/article/details/90715730)
+ [以android系统为例的OTA升级](https://www.cnblogs.com/startkey/category/1416503.html)