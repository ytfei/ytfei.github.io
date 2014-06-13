---
layout: post
title: Android 源码下载及编译
categories: [android]
date: 2014-06-10 10:24:10 +0800
---

主要是记录下我的源码获取及编译过程。

我的环境：

* VMWare Workstation 10.0.1 build-1379776
* Ubuntu 12.04.4 LTS (64-bit)
* Samsuang Galaxy Note II (N7100)

Android 版本：CyanogenMod 11.0 (Based on Android 4.4)

<!--more-->

## 准备网络环境

网上关于 Android 如何获取及编译的内容都很多了，过程相对简单。
只是由墙的问题，下载代码还是有点麻烦。

翻墙是第一道工序了。

我使用的是公司的海外的服务器作为代理。
如果没有这个条件，则可以在 AWS 上申请一台临时机器用用。
第一次使用的可以免费得到一台 micro 机型一年的使用期。当作海外代理还是可以用用的，
只是网络不是特别好。凑合吧。

{% highlight bash %}
ssh -D localhost:1080 my-server
{% endhighlight %}

然后安装 Privoxy 作为 Http 的代理 （因为有些程序不支持 socks 代理）。

{% highlight bash %}
sudo apt-get install privoxy

# 配置 Privoxy
forward-socks5   /               127.0.0.1:1080 .
forward                            .github.com     . 
forward                            .baidu.com     . 
forward          192.168.*.*/     .
forward          10.*.*.*/        .
forward          127.*.*.*/       .
{% endhighlight %}

Privoxy 是作为系统服务，默认启动的。它的地址是： `http://127.0.0.1:8118` 。

到这里，你就有了一个 socks 代理 和一个 http 代理了。

设置环境变量：

{% highlight bash %}
# for git
export http_proxy=http://localhost:8118

# for repo (google android source code access)
export HTTP_PROXY=http://localhost:8118
export HTTPS_PROXY=http://localhost:8118
{% endhighlight %}

## 虚拟机准备

CM 的所有代码总计大概有 24G 的样子，编译完成后，所有的空间大约为 60G。

因此为了保证编译的正常进行，至少60G的硬盘空间是需要的。我建议单独为 Android 的代码准备一个虚拟盘，
这样代码的移动和备份都会方便很多。

之前有文章介绍[如何添加虚拟磁盘](http://codingme.com/blog/android/linux/2014/01/10/Add-new-disk-to-VM/)。

为了编译 Android，以下的依赖包是必须的：

{% highlight bash %}
bison build-essential curl flex git gnupg gperf libesd0-dev libncurses5-dev libsdl1.2-dev libwxgtk2.8-dev libxml2 libxml2-utils lzop openjdk-6-jdk openjdk-6-jre pngcrush schedtool squashfs-tools xsltproc zip zlib1g-dev
{% endhighlight %}

64位环境额外需要：

{% highlight bash %}
g++-multilib gcc-multilib lib32ncurses5-dev lib32readline-gplv2-dev lib32z1-dev
{% endhighlight %}

我为 Android 准备的虚拟磁盘为 `/dev/sdg1`。

{% highlight bash %}
cd ~
mkdir -p ~/codebase/opensource/CyanogenMod
sudo mount /dev/sdg1 ~/codebase/opensource/CyanogenMod
mkdir -p ~/codebase/opensource/CyanogenMod/android/system

mkdir ~/bin
mkdir -p ~/android/
cd ~/android && ln -s ~/codebase/opensource/CyanogenMod/android/system system
{% endhighlight %}

## 下载代码

下载 repo 脚本：

{% highlight bash %}
# 为 wget 设置代码
sudo cat >> /etc/wgetrc <<'EOF'
http_proxy=http://localhost:8118
https_proxy=http://localhost:8118
EOF

wget https://storage.googleapis.com/git-repo-downloads/repo -O ~/bin/repo
chmod u+x ~/bin/repo
{% endhighlight %}

初始化：

{% highlight bash %}
cd ~/android/system
repo init -u git://github.com/CyanogenMod/android.git -b cm-11.0
{% endhighlight %}

同步代码！这一步是相当漫长的，所用时间取决于你的网速了。
想想一下要下载 20 多G文件的样子。我大概是用了 7 个小时。
而且中间总是因为网络问题导致同步过程反复中断。

从网上找到一个避免中断的选项，`-f` 。
如果上面的环境都配置成功了，把机器开一晚上应该就没问题了。

{% highlight bash %}
repo sync -f -j10
# -f | --force-broken (continue sync even if a project fails to sync)
# -j N thread number to poll
{% endhighlight %}

## 手机准备

在下载代码的过程，我们还有另外一个环境要准备——手机。

你得先把你准备的 N7100 刷一个 CM 版本，最好是和你要编译的版本兼容。

在这里可以找到你需要的 CM 版本：[N7100 CM](http://download.cyanogenmod.org/?device=n7100)

我选择的是 cm-11-20140505-SNAPSHOT-M6-n7100.zip 。

对了，刷机我用的是刷机精灵，直接偷懒了。

之所以要先刷一个 CM，是因为我们后面编译时，要从现有的手机上提取一些驱动文件等数据。

## 开始编译

代码下载完成后，开始编译。

{% highlight bash %}
$ cd ~/android/system/vendor/cm
$ ./get-prebuilts

## 初始编译环境
$ cd ~/android/system
$ source build/envsetup.sh
$ breakfast n7100
{% endhighlight %}


`breakfast n7100` 会下载一下机型相关的数据，kernel 等。你可能会在这一步失败。没关系，先继续。

{% highlight bash %}
cd ~/android/system/device/samsung/n7100
./extract-files.sh
{% endhighlight %}

再执行一遍 `breakfast n7100`，成功了！


{% highlight bash %}
# 据说 Android 只能在 32位的环境编译，为了在64位的机器上工作，
# 需要设置以下环境变量
export BUILD_HOST_32bit=1
{% endhighlight %}

**可选设置**
设置编译缓存，以提高编译速度。 这个很占空间，我磁盘不足，没有使用。

**注意，我之前给出的空间量 60G 是没有包含 cache 的。**

{% highlight bash %}
export USE_CCACHE=1
prebuilts/misc/linux-x86/ccache/ccache -M 50G
{% endhighlight %}

为了防止编译 Java 代码时内存不够，可以防御性地做以下修改：

{% highlight bash %}
修改 system/build/tools/releasetools/common.py
将 java -Xmx2048m 改为 java -Xmx1024m 或者 java -Xmx512m
{% endhighlight %}


开始编译！

{% highlight bash %}
croot
brunch n7100
{% endhighlight %}

我不太清楚花了多少时间，反正电脑放在办公室里一晚上，第二天早上上班时发现已经完成。

编译过程电脑发执厉害，CPU 估计全程 200% ，所以最好把笔记本放在散热好的位置，
免得死机或者重启，又得重来。