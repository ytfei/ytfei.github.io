---
layout: post
title: 编译Android源码
categories: [android, linux]
date: 2014-01-11 16:40:00 +0800
---

## 编译环境

* 主机：Ubuntu 12.04 LTS (VM Ware 32bit)
* Android: CyanogenMod 10.2
* 机型：HTC One X (endeavoru, S720e)

## 编译过程

首先，我使用的环境是半自动的——虽然官方网站上提供了了`HTC One X`的ROM，但是其编译的手册却与源码不相符。

在从手机中提取文件这一步，`extract-files.sh` 已经不存在了。

### 参考资料

* [Build CyanogenMod for HTC One X](http://wiki.cyanogenmod.org/w/Build_for_endeavoru)
* [How To Port CyanogenMod Android To Your Own Device](http://wiki.cyanogenmod.org/w/Doc:_porting_intro)
