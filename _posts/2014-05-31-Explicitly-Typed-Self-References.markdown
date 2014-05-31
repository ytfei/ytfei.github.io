---
layout: post
title: Scala 约束子类类型
categories: scala
date: 2014-05-31 11:20:00 +0800
---

通过子类类型的约束（在多态继承的情况），对维持软件模块的可扩展性很有意义。
在这里，Scala 提供了一种机制，可以让接口预先指定 `this` 的类型。

以下是通过一个例子来说明这点。

<!--more-->

假定要开发一个图状的结构，一个图（Graph）是由各种节点 (Node) 以及连接这些节点的线 (Edge) 组成。

{% highlight scala %}
abstract class Graph {
  type Edge
  type Node <: NodeIntf

  abstract class NodeIntf {
    def connectWith(node: Node): Edge
  }

  def nodes: List[Node]
  def edges: List[Edge]
  def addNode: Node
}
{% endhighlight %}

在 `Graph` 中，我们定义两个抽象类型，并对 `Node` 做了一定的约束——要求其实现具备 `connectWith` 方法。
同时还预留出抽象函数 `addNode` 用于向 `Graph` 类型默认类型的节点。

用户可以通过扩展 `Graph` 接口来实现自己的图形结构，比如实现具有颜色的节点，或者各式类型的线条等。

下面是一种可能的实现：

{% highlight scala %}
abstract class DirectedGraph extends Graph {
  type Edge <: EdgeImpl

  class EdgeImpl(origin: Node, dest: Node) {
    def from = origin
    def to = dest
  }

  class NodeImpl extends NodeIntf {
    def connectWith(node: Node): Edge = {
      val edge = newEdge(this, node)
      edges = edge :: edges
      edge
    }
  }

  protected def newNode: Node
  protected def newEdge(from: Node, to: Node): Edge

  var nodes: List[Node] = Nil
  var edges: List[Edge] = Nil
  def addNode: Node = {
    val node = newNode
    nodes = node :: nodes
    node
  }
}
{% endhighlight %}

`DirectedGraph` 对 `Graph` 做了一定扩展，但是只对部分接口进行了实现，
因此我们可以在它的基础继续衍生。`DirectedGraph` 定义了 `Edge` 类型的上限，
同时也实现了 `NodeIntf` 接口。另外，由于我们需要在 `Graph` 接口内创建 `Node` 和 `Edge` 实例，
`DirectedGraph` 又增加了两个额外的工厂方法 `newNode` 和 `newEdge`。

但是，这里我们遇到了一个问题，即 `NodeImpl` 的实现是有问题的。虽然 `Graph` 
指定了 `Node` 的上限，但并不表示实现了 `NodeIntf` 接口的都是 `Node` 类型。这就导致 `NodeImpl` 
中的操作 `val edge = newEdge(this, node)` 是非法的——因为 `this` 不一定是 `Node` 类型。

这时我们就需要对子类的类型进行约束，如下：

{% highlight scala %}
abstract class DirectedGraph extends Graph {
  ...
  class NodeImpl extends NodeIntf {
    self: Node => // 约束子类的类型

    def connectWith(node: Node): Edge = {
      val edge = newEdge(this, node)  // now legal
      edges = edge :: edges
      edge
    }
  }
  ...
}
{% endhighlight %}

稍加改变后，`NodeImpl` 的 `this` 就具备的 `Node` 的特性。

这种约束非常有用，尤其是需要回调子类方法时，可以像下面这样（一个没用的例子:)）。

{% highlight scala %}

trait P {
	self: S =>

	def methodA = {
		s"Wrapped in parent: ${self.methodB}"
	}
}

{% endhighlight %}

回到正题，下面接着给出一个 `DirectedGraph` 的具体实现：

{% highlight scala %}
class ConcreteDirectedGraph extends DirectedGraph {
  type Edge = EdgeImpl
  type Node = NodeImpl

  protected def newNode: Node = new NodeImpl
  protected def newEdge(f: Node, t: Node): Edge =
    new EdgeImpl(f, t)
}
{% endhighlight %}

> **NOTE:** 这里可以实例化 `NodeImpl` 是因为上文已经指定了 `Node` 的类型，
> `type Node = NodeImpl`，因此 `NodeImpl` 本身也是 `Node`。

下面是一个简单的测试，将上面的代码窜起来：

{% highlight scala %}
object GraphTest extends App {
  val g: Graph = new ConcreteDirectedGraph
  val n1 = g.addNode
  val n2 = g.addNode
  val n3 = g.addNode
  n1.connectWith(n2)
  n2.connectWith(n3)
  n1.connectWith(n3)
}
{% endhighlight %}

原文: http://docs.scala-lang.org/tutorials/tour/explicitly-typed-self-references.html