---
layout: single
related: false
title:  Android LCD背光驱动节电技术LABC/CABC
date:   2020-01-05 20:32:00 +0800
categories: android
tags: display
toc: true
---

# 1. LCD背光驱动节电技术LABC/CABC

> 手机屏幕大部分是LCD（还有OLED屏幕），而手机的部分电量就是LCD背光消耗的。随着分辨率/尺寸的增大，LCD的背光驱动电路也越来越复杂。而高分辨率、高显示颜色、大尺寸的LCD，需要大的背光系统、大的TFT-LCD 面版、高运算速度的驱动IC，这些都造成了高的功率消耗。  
> 主要了解一下背光驱动节电技术CABC的概念和功能。[参考博客](https://blog.csdn.net/sinat_30545941/article/details/72874775)

<!--more-->
***

# 2. *OEM和ODM

**Note:**

1. OEM：自主加工，英文全称Original Equipment Manufacturer，即原设备生产商。原始设备生产商(OEM)是指拥有自己的产品或产品理念，但有时会为了开发和/或制造这些产品而购买某些服务的公司。
2. ODM：自主设计，即ORIGINAL DESIGN MANUFACTURER，意为“原始设计制造商”，是指一家公司根据另一家公司的规格来设计和生产一个产品。例如，计算机公司如HP公司可能会就其想推向市场的一款笔记本电脑作出具体规格。它们会具体地列明产品的外观要求，如屏幕的尺寸和技术要求、输入/输出端口、键盘的前倾度、电脑包的外形和颜色、扬声器的位置等。它们还通常会具体列明对产品的主要内部细节如CPU或视频控制器的规格要求。但是，它们并不设计图样，不具体列明电源用的交换晶体管的型号，也不对背光变流器频率加以选择。这些都是ODM的工作。ODM根据计算机公司提出的规格要求来设计和生产笔记本电脑。有时候，ODM也可根据现有样品来生产。ODM方式往往更加注重合作，而在OEM的情形下，购买方对产品的具体规格基本不参与意见。
3. OBM：自主品牌

**OEM和ODM的区别:**

OEM和ODM两者最大的区别不单单是名称而已。OEM产品是为品牌厂商度身订造的，生产后也只能使用该品牌名称，绝对不能冠上生产者自己的名称再进行生产。而ODM则要看品牌企业有没有买断该产品的版权。如果没有的话，制造商有权自己组织生产，只要没有企业公司的设计识别即可。

在工业社会中，OEM和ODM可谓司空见惯。因为出于制造成本、运输方便性、节省开发时间等方面的考虑，知名品牌企业一般都愿意找其他厂商OEM或ODM。在找别的企业进行OEM或ODM时，知名品牌企业也要承担不少责任。毕竟产品冠的是自己的牌子，如果产品质量不佳的话，少则有顾客找上门投诉，重则可能要上法庭。所以，品牌企业在委托加工期间肯定会进行严格的质量控制。但代工结束后，质量不敢保证。故此，当有的商家告诉你某件产品的生产商是某大品牌的OEM或ODM产品时，绝不要相信其质量就等同于该品牌。你唯一能够相信的，是这家制造商有一定的生产能力。

***

# 3. 背光节电技术

显示屏在移动设备里一直的是耗电大户。目前手机背光节电技术，即对应性背光控制技术（Adaptive Brightness Control- ABC），主要有下面2种：

1. LABC：(LightAdaptive Brightness Control) `环境光侦测适应背光控制`。根据环境光的变化来控制背光亮度。需要一个光传感器，感应环境光强。
2. CABC：(ContentAdaptive Brightness Control）`显示内容对应背光控制`。根据显示内容来调节背光和gamma值，从而降低了背光LED的功耗。其中C是内容的意思，驱动IC新增了一个内容分析器电路。

# 4. LABC

LABC技术需要搭配光传感器实现，主机端处理器读取光感数值，然后处理器对数值进行处理，直接控制PMIC(MT6329)输出PWM控制背光的亮度。在比较暗光线下，降低背光达到省电效果

# 5. CABC

CABC功能需要在LCD驱动IC内新增一个内容分析器(imagecontent analyzer)电路，当手机处理器传送了一张图片数据到驱动IC，内容分析器会计算并统计图片的数据后依据设定与算法自动的将其灰阶亮度提高30%（此时图片变亮），再将背光亮度降低30%（此时图片变暗）。由于我们事先已经将图片经过分析器电路补偿亮度，因此使用者可以得到与原先电路相差无几的显示效果，但减少了30%的背光功耗。

简单来说，CABC功能就是根据`显示内容`来降低背光，然后通过调`节gamma`(gamma越高灰度越低图像越暗)来补偿显示亮度。CABC就是通过增加内容灰阶标准同时降低背光亮度来达到省功耗的目的。

CABC主要有四种状态：

1. Off Mode，CABC功能全部关闭；
2. UI Image Mode，优化显示UI图片时的功耗，尽可能的保证图片质量的同时可省10%的功耗；
3. Still Image Mode，优化显示静态图片时的功耗，该模式下图片质量损耗在可接受的范围内，同时可省30%的功耗；
4. Moving Image Mode，优化显示动态图片时的功耗，该模式下会最大限度的降低功耗，但是会带来图片质量的损耗，可省30%+的功耗。

自然对应Off Mode，标准对应UI Mode，照片对应UI Mode，电影对应Still Image Mode。用户可根据实际情况自行选择。

三种模式省电级别依次降低: `UI mode < Movie mode < Still mode`， 也就是说Still mode是最省电的模式。

## 5.1. 工作流程

工作流程如下：

1. 使能PMIC(MT6329)CABC功能;
2. 设置LCD驱动IC的相关配置(使能CABC和配置gamma参数，需要FAE协助)；
3. CABC模块分析显示内容输出LED_PWM信号给PMIC，PMIC通过一定算法控制driver模块BL_DRV信号的输出波形；
4. 预期结果是背光亮度降低，LCD驱动IC降低gamma值以补偿屏幕亮度。

CABC模块分析显示内容输出`PWM波形`，占空比越大，表示需要输出的电流越大。
