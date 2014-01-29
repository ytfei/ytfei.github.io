---
layout: post
title: Appium环境抢建(for web browser test)
categories: [android, appium, test]
date: 2014-01-29 11:27:30 +0800
---

[Appium](http://appium.io/)是Android平台上一个测试框架。

> 应用环境：
>
> * Ubuntu 12.04 LTS
> * HTC One X (endeavoru, S720e)

# Android SDK

请参考[SDK环境](http://developer.android.com/sdk/installing/bundle.html)，这里就不多说了。

# Appium

1. 安装`nodejs`

{% highlight bash %}
apt-get install nodejs

# 或者通过nodejs源码编译，这样可以使用最新的代码
cd ~/downloads
wget http://nodejs.org/dist/v0.10.25/node-v0.10.25.tar.gz
tar -zxf node-v0.10.25.tar.gz
cd ode-v0.10.25
./configure --prefix=/usr/local/node
make && make install

# edit ~/.bashrc and add node to your PATH env
{% endhighlight %}

2. 安装`appium`

{% highlight bash %}
npm install -g appium  # install appium as a global app
{% endhighlight %}

3. 配置手机

手机需要是已经root过的！

{% highlight bash %}
adb shell
su
chmod 777 /data/local
{% endhighlight %}

另外，也要确保你手机上安装了最新的chrome浏览器!

> **Note:**
>
> 这步是必需的，否则后面会发生无法启动浏览器的异常。

3. 下载&运行测试项目

{% highlight bash %}
# 下载项目
git clone git@github.com:ytfei/appium_chrome_demo.git

cd appium_chrome_demo
npm install  # 安装依赖包

# 启动appium
appium -g appium.log &

# 开始测试
node web.js
{% endhighlight %}
