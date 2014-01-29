---
layout: post
title: git merge 之 squash
categories: [android, linux]
date: 2014-01-11 17:26:30 +0800
---

看CM源码时，发现历史记录里有很多`squash`，于是google了解了一下。

Git相对于CVS和SVN的一大好处就是merge非常方便，只要指出branch的名字就好了，如：

{% highlight bash %}
$ git merge another
$ git checkout another
# modify, commit, modify, commit ...
$ git checkout master
$ git merge another
{% endhighlight %}

但是，操作方便并不意味着这样操作就是合理的，在某些情况下，我们应该优先选择使用`--squash`选项，如下：

{% highlight bash %}
$ git merge --squash another
$ git commit -m "message here"
{% endhighlight %}

`--squash` 选项的含义是：本地文件内容与不使用该选项的合并结果相同，但是不提交、不移动HEAD，因此需要一条额外的commit命令。其效果相当于将another分支上的多个commit合并成一个，放在当前分支上，原来的commit历史则没有拿过来。

> **Note:**
>
> 判断是否使用`--squash`选项最根本的标准是，待合并分支上的历史是否有意义。

如果在开发分支上提交非常随意，甚至写成微博体，那么一定要使用`--squash`选项。版本历史记录的应该是代码的发展，而不是开发者在编码时的活动。

只有在开发分支上每个commit都有其独自存在的意义，并且能够编译通过的情况下（能够通过测试就更完美了），才应该选择缺省的合并方式来保留commit历史。

[这里是原文](http://alpha-blog.wanglianghome.org/2010/08/05/git-merge-squash/)
