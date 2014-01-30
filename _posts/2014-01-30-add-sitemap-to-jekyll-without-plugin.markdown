---
layout: post
title: 在JekyII中添加站点地图
categories: [android, appium, test]
date: 2014-01-30 10:05:30 +0800
---

按以下步骤，不需要使用插件，便可以为自己的JekyII添加站点地图。

<!--more-->

假设你的站点在目录`/data/site`目录中。

{% highlight bash %}
cd /data/site
touch sitemap.xml
{% endhighlight %}

将下面的内容添加到`sitemap.xml`文件中。

{% gist 8701413 %}

这个模版文件引用了如下的几个参数：

{% highlight yaml %}
# add 'url' to '_config.yml', 值应该是你的站点名
# in _config.yml
url: http://codingme.com/

# 下面的参数则是每篇文章中可选的，你可以把它们加了文章的头部，
# 即你定义'layout'等参数的地方。
sitemap:
  lastmod: 2014-01-30
  priority: 0.7
  changefreq: daily
{% endhighlight %}

> **Note:**
>
> 上面所提及的参数都是**可选**的。其中`url`的默认值是`/`，即根路径。
> 如果你不将其替换为你的站点路径，则生成的站点地图(sitemap.xml)在被提交到搜索引擎时，
> 会被认为是无效的。
> 
> 至于`sitemap`的几个参数可以不加，或者在你修订文章时再加也可以。
