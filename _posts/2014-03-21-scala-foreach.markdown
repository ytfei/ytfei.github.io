---
layout: post
title: scala foreach 使用
categories: scala
date: 2014-03-21 13:28:44 +0800
---


今天看Spark的代码，发现`RDD`并没有实现任何集合类的接口，但是可以用于`for`语句中。
通过观察，我发现使用`for`语句的类结构都有相似的地方，如都有`foreach(f: T => Unit)`方法。

我猜这个方法与`for`语句应该是有关联的。以下是验证。

<!--more-->

直接上代码吧，清晰简单。

{% highlight scala %}
// demo.scala
class X[T](val e:T) {
  def foreach(f: T => Unit) {
    f(e)
  }
}

class Y[T](val e:T) {
  def each(f: T => Unit) {
    f(e)
  }
}

{% endhighlight %}

{% highlight bash %}
evans@ubuntu:~/tmp/$ scala -i demo.scala 
Loading x.scala...
defined class X
defined class Y

Welcome to Scala version 2.10.3 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_51).
Type in expressions to have them evaluated.
Type :help for more information.

scala> val x = new X[Int](10)
x: X[Int] = X@498f5e80

scala> val y = new Y[Int](20)
y: Y[Int] = Y@11fa739a

scala> for(e <- x) println(e)
10

scala> for(e <- y) println(e)
<console>:10: error: value foreach is not a member of Y[Int]
              for(e <- y) println(e)
                       ^

scala> exit
{% endhighlight %}

由此可以推断，凡是实现了`def foreach[T](f: T=> Unit)`方法的，都可以使用`for`语句。

> **Note**
> 
> 以上内容纯属猜测 :)

