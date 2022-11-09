---
layout: single
related: false
title:  ubuntu输入法无法选择候选文字
date:   2019-08-29 22:28:00
categories: linux
tags: linux
toc: true
---

> ubuntu输入法无法选择候选文字

# 1. 解决方法

```shell
#删除配置文件
rm -rf ~/.cache/ibus/libpinyin
ibus-daemon -drx
ibus-daemon -drx
```
<!--more-->