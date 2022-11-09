---
layout: single
related: false
title:  nReal搭建Samepl APK
date:   2019-09-30 21:52:00
categories: VR
tags: nReal
toc: true
---

> nrsdk是nreal开发混合现实体验的平台。使用简单的开发过程和高级api，nrsdk提供了一组强大的mr特性，并使nreal眼镜能够了解真实世界。

# 1. 概述

nrsdk为开发者提供了五个核心特性：
1. 空间计算使眼镜能够跟踪它们相对于世界的实时位置，并了解周围的环境，例如检测和跟踪平面和图像。
2. 优化的渲染自动应用于应用程序并在后端运行，以最小化延迟并减少抖动，增强总体用户体验。
3. 多模态交互为不同的用例提供了交互的直观选择。
4. 提供了开发工具，以便您可以更好地开发和调试应用程序。
5. 第三方集成是通过为第三方sdk提供数据来实现的，这允许您充分利用nreal light的硬件功能并构建功能强大的mr/ar应用程序。

# 2. 开发工具包

开发混合现实应用程序需要一个nreal开发工具包。开发工具包由一对nreal光学眼镜、nreal计算单元和nreal光学控制器组成。如果没有，请在这里注册nreal开发工具包！
到目前为止，你还不能直接在android手机上开发应用程序。安卓手机开发将于2020年初推出。

# 3. 选择开发平台

nrsdk支持许多最流行的开发环境。通过这些功能，您可以构建全新的mr体验，或者使用mr功能增强现有的原生android应用程序。
Unity （Support Unity 2018.2.X） Android Native (to be released) Unreal (to be released)

***

# 4. 与Android本机应用程序兼容

nreal眼镜现在与android原生应用程序兼容，这意味着只要应用程序安装在设备上，用户就可以通过眼镜查看所有应用程序活动。在你这边什么都不需要改变。要使2d应用程序更具沉浸感和三维感，可以使用nrsdk在现有应用程序中添加mr功能或3d虚拟对象。

***

# 5. 功能

## 5.1. 空间计算

nreal眼镜使用各种传感器和相机，以建立对周围环境和用户本身的复杂理解，创造身临其境的体验，无缝融合数字世界和现实世界。

***

# 6. HelloMR APP

## 6.1. 硬件清单

1. 一个nreal计算单元（把它想象成一个没有屏幕的android手机，所以所有的开发过程都将非常类似于移动应用程序开发）。
2. 一副自然光眼镜
3. 没有nreal设备？注册nreal开发工具包！或者尝试仿真器在没有nreal眼镜和计算单元的情况下引导nreal应用程序功能。
4. 连接nReal计算单元和PC的USB-C电缆
5. 不需要Wi-Fi连接。但是，可以使用Wi-Fi Android调试桥（ADB）连接进行调试和测试。

## 6.2. 软件清单

1. Unity 2018.2.x或更高版本，支持Android构建
2. 下载Unity 1.1的nrsdk(sdk作为nrsdkforunity_1.1.unitypackage下载)
3. android sdk 7.0（api级别24）或更高版本，使用android studio中的sdk管理器安装

## 6.3. 创建统一项目

1. 打开Unity并创建一个新的3D项目。
2. set  player settings>other settings>scriptping runtime version to.net 4.x等效版本
3. 为unity导入nrsdk
3.1. 选择“资源>导入包>自定义包”。
3.2. 选择您下载的nrsdkforunity_1.1.unitypackage。
3.3. 在“导入包”对话框中，确保选中了所有包选项，然后单击“导入”。

在unity项目窗口中，通过选择assets>nrsk>demos>hellomr找到hellomr示例应用程序。

## 6.4. 配置生成设置

1. 转到“文件>生成设置”。
2. 选择android并单击switch platform。
3. 在“生成设置”窗口中，单击“播放器设置”。
4. 在Inspector窗口中，按如下方式配置播放机设置：(https://developer.nreal.ai/develop/unity/android-quickstart)

## 6.5. 连接到nReal设备(开发工具包)

在计算单元上启用开发人员选项和USB调试。android调试桥（adb）作为默认设置启用，不需要手动设置）。
将计算单元连接到Windows PC。

## 6.6. 建立并运行

1. 在Unity Build设置窗口中，单击Build。在构建成功后，通过wifi android调试桥（adb）安装应用程序。
2. 断开电脑与电脑的连接，然后将其连接到眼镜上。
3. 如果这是您第一次运行此应用程序，则需要使用某些工具（如scrcpy）对该应用程序进行身份验证。
4. 与nReal Light控制器一起启动应用程序。有关如何使用nReal Light控制器的说明，请参阅控制器指南。
5. 四处移动，直到nrsdk找到一个水平面，检测到的平面将被绿色网格覆盖。
6. 单击触发器按钮在其上放置nReal徽标对象。
7. （可选）使用android logcat查看记录的消息。我们建议使用WiFi Android调试桥（ADB）连接到您的PC，这样您就不必在大多数时间通过数据线连接。

***

# 7. Sample APP(立体方块)

## 7.1. scene设置

1.在SampleScene删除主摄像头；

2.将Assets -> NRSDK -> Prefabs -> NRCameraRig拖拽到SampleScene中；

3.将Assets -> NRSDK -> Prefabs -> NRInput拖拽到SampleScene中；

4.将Assets -> NRSDK -> Emulator -> Prefabs -> NRTrackableImageTarget拖拽到SampleScene;

在其中可以修改场景图像（Image Target）

5.在SampleAPP右侧窗口中右击Create -> 3D -> Object -> Cube，创建立方体

将Scale均修改成0.25

6.右击Create Empty,在Inspector中Add Component

(1)Script设置为TrackableFoundTest

(2)Observer设置为NRTrackableImageTarget

(3)Obj设置为Cube

7.打开Assets -> NRSDK -> Emulator -> Scripts -> TrackableFoundTest,编辑源文件C#，添加Update函数，增加每次点击切换立方体颜色

```cpp
using NRKernal;
...
   private void Update()
    {
        if (NRInput.GetButtonDown(ControllerButton.TRIGGER))
            Obj.GetComponent<Renderer>().material.color = new Color(Random.value, Random.value, Random.value);
    }
```

8.File -> BuildSettings添加SampleAPP，然后选择Android，最后Building

9.生成APK，安装full screen black

10.或者在Unity上方点击Play按钮使用Emulator查看，常用按键操作：

```shell
WASD控制前后左右；
Space+鼠标表示头部旋转；
SHIFT+鼠标移动模拟nReal控制器的旋转（3DOF控制器）；
单击鼠标左键模拟控制器触发器的单击（此处会触发Update函数变化颜色）；
单击鼠标右键以模拟控制器主页按钮的压力；
单击鼠标滚轮按钮以模拟控制器的APP按钮的压力；
使用箭头键模拟在控制器的触摸板上滑动；
```
