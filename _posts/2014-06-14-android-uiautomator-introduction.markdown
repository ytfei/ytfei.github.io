---
layout: post
title: Android Ui Automator 入门
categories: [android]
date: 2014-06-14 15:11:42 +0800
---

最近 Google 的 Android 官网访问不畅啊。下载了 Android 的离线文档，在本地打开还是非常慢，
原来是文档中都有引用了一个托管的 CSS 文件。

好吧，先处理一下文档：

cd /opt/adt-bundle-linux-x86_64-20131030/sdk/docs
find ./ -name "*.html" | xargs sed -i '/googleapis.com/d'