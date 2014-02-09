---
layout: post
title: Netty4 中的内存管理
categories: java netty
date: 2014-02-09 17:48:44 +0800
---

在Netty4中引入了新的内存管理机制极大地提升其性能，本文将对该内在管理机制进行剖析。

这里[有篇文章](http://www.infoq.com/news/2013/11/netty4-twitter)讲述了在推特(Twitter)内部
使用Netty的状况以及Netty4所带来的性能收益。

<!--more-->

在分析Netty4的`PooledByteBufAllocator`之前，我们最好先认识一下[jemalloc](http://www.canonware.com/jemalloc/)。
Netty在4.0之前的版本已经尝试过通过优化内存管理的方式来提高性能（如果我没有记错的话），但4.0中的改进则特别
显著。在这个版本中，其内存管理实现主要是参考了`jemalloc`。

# jemalloc

**jemalloc** 是由Jason Evans在FreeBSD项目中引入的，其主旨是为了提升在并发环境下内存的分配效率。说白了就是替代
`malloc`。malloc之所以没有照顾到并发环境，那是由于在那个时代并发还只在理论，未曾普及。而现在则是多核的天下，连
手机都动则2、4核，甚至于8核了。与jemalloc齐名的还有Google的[tcmalloc](https://code.google.com/p/gperftools/)，其
实现与jemalloc多少也有点相似，这里不做介绍。

## jemalloc的理念

我们以买火车票为例，来简单地说明一下jemalloc与malloc的区别。原来的malloc，相当于只有一个售票窗口的售票大厅，
而jemalloc则在同一个售票大厅里面适量地增加的窗口。当然，火车票的总量(即内存大小)是不变的，买票的人相当于线程了。
说起来这也是很自然的事情的。

> 在这里，一个售票窗口就是相当于一个**Arena**。

Arena则按页(Page)来的管理内存，也就是说，一张车票就相当于一页。（后面就不太适合用火车票的例子了）。

同时，jemalloc还根据所请求的内存大小，对其进行分类。如下图：

![jemalloc allocation size category](http://farm4.staticflickr.com/3749/12402575993_30e006725c_o.png)

默认情况下，Page的大小为4KB。如图，有三类，small、large和huge。small类的内存请求都属于一个内存页之内
（没有半张车票出售:(）。另外，在small类里面，又分了三个子类，分别是Tiny、Quantum-Spaced和Sub-page。
这几个概念都在Netty中得到应用。

每个线程都与某个Arena绑定在一起，线程采用round-robin算法来绑定到某个Arena。

> 这里有个问题，就是与某个Arena相关的一批线程使用内存资源过快，导致该Arena的内存资源全部消耗殆尽，
> 而其它的Arena又有盈余，这时怎么处理？能否借用。
> 
> 目前还没有对jemalloc本身的实现做过多的研究。

通过对jemalloc有个简单的了解后，我们再来看看，Netty4是如何借鉴jemalloc的经验的。需要注意的是，
jemalloc的实现，要远比Netty4复杂，一个是系统层的，一个则是应用层的。但两者的思想是相通的。

# Netty4中的内存管理

以下内容所参考的代码是`netty-4.0.15.Final`。

Netty4中，内存池管理的入口点是`PooledByteBufAllocator`。在该实现中，除了Arena和Page之外，
还一个Chunk的概念。一个Arena由多个Chunk组成，而Chunk则由N个Page组成。

默认配置中，

{% highlight java %}
// 默认页面大小为8KB
int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192); 

// 默认的Chunk大小为16MB，即由2048个页面组成
int defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;

// 默认情况下，一个Arena由3个Chunk组成，且所有Arena（就同一类而言）所占用的内存总数，
// 不能超过可用内存的一半(50%)。

DEFAULT_NUM_HEAP_ARENA = Math.max(0,
        SystemPropertyUtil.getInt(
                "io.netty.allocator.numHeapArenas",
                (int) Math.min(
                        runtime.availableProcessors(),
                        Runtime.getRuntime().maxMemory() / defaultChunkSize / 2 / 3)));
DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
        SystemPropertyUtil.getInt(
                "io.netty.allocator.numDirectArenas",
                (int) Math.min(
                        runtime.availableProcessors(),
                        PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));

{% endhighlight %}

根据以上代码，我们发现，Arena是分为两类的，一类是基于堆；另一类则是Direct内存（直接向操作系统请求的）。
具体使用哪一类，默认是根据所使用的虚拟机而定，优先使用Direct类型（如果可用的话）。判断的条件是，
看能否访问JVM的`Unsafe`类。具体可以参考`PlatformDependent.hasUnsafe()`方法。

> 这里提到的两类不会被同时使用的！


另外，对于内存请求大小的分类，这里也有一些变动。对于所请求内存的大小N

{% highlight java %}

if N < PAGE_SIZE:
	if N < 512:
		return TINY;
	else
		return SMALL;
elif N > CHUNK_SIZE:
	return HUGE;
else
	return LARGE;

{% endhighlight %}

## 内存管理

现在，我们来仔细地看看内存分配的过程。

{% highlight java %}

PooledByteBufAllocator alloc = new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());
ByteBuf buff = alloc.ioBuffer(N);

{% endhighlight %}

假设在当前的JVM环境中，Direct内存可用的。那当我们尝试去申请N字节的内存时，其实现流程如下：

1.  获取一个ByteBuf对象，但是该对象未与任何可用内存空间关联。
2.  检查N的大小，并对N进行修正。。
3.  分配内存。


对N的修正：

{% highlight java %}
    private int normalizeCapacity(int reqCapacity) {
        if (reqCapacity < 0) {
            throw new IllegalArgumentException("capacity: " + reqCapacity + " (expected: 0+)");
        }
        if (reqCapacity >= chunkSize) {
            return reqCapacity;
        }

        if ((reqCapacity & 0xFFFFFE00) != 0) { // >= 512
            // Doubled

            int normalizedCapacity = reqCapacity;
            normalizedCapacity |= normalizedCapacity >>>  1;
            normalizedCapacity |= normalizedCapacity >>>  2;
            normalizedCapacity |= normalizedCapacity >>>  4;
            normalizedCapacity |= normalizedCapacity >>>  8;
            normalizedCapacity |= normalizedCapacity >>> 16;
            normalizedCapacity ++;

            if (normalizedCapacity < 0) {
                normalizedCapacity >>>= 1;
            }

            return normalizedCapacity;
        }

        // Quantum-spaced
        if ((reqCapacity & 15) == 0) {
            return reqCapacity;
        }

        return (reqCapacity & ~15) + 16;
    }
{% endhighlight %}

说到内存的分配，则需要对页的管理有一定的了解。

### 内存页的管理模型

Netty4使用了二叉树来管理每一个内存页。假设一个Chunk是由连续的N块内存页组成，
则Netty4使用一个长度为`2N`的整型数组来记录和管理每一个内存页的使用情况。

{% highlight java %}
int[] memoryMap = new int[N << 1]
{% endhighlight %}

假设N=4，则下图，基于`memoryMap`的二叉树的结构。每个结点的序号为数组的索引值。其中4、5、6、7是这四个叶子节点，
分别用来记录4个内存页的使用状态。而2、3是用来表示其管辖的内存页的使用状态。节点1则管理整个Chunk的状态。

![基于一维数组的二叉树](http://farm3.staticflickr.com/2805/12403894375_2ee7cf93a4_o.png)

内存页的状态有四种：

{% highlight java %}
private static final int ST_UNUSED = 0;  // 初始状态，未使用

private static final int ST_BRANCH = 1;  // 只对非叶子节点有效，表明该节点下某一部分叶子节点被使用

// 被使用，如果是叶子节点，则表明对应的内存页已经分配出去了
// 如果是非叶子节点，则表明该节点所管辖的内存页已经全部分配出去了
private static final int ST_ALLOCATED = 2; 

// 只对叶子节点有效，表示内存页里面的一部分被分配出去了。这种情况属于Tiny类型的内存请求。
private static final int ST_ALLOCATED_SUBPAGE = ST_ALLOCATED | 1;
{% endhighlight %}

### 内存页的分配

内存页的分配有两种情况，有多页分配和页内分配。这里我们先说多页（1个内存页以上）的分配。
当请求的内存大小大于一个内存页时，`PooledByteBufAllocator`会通过遍历`memoryMap`来，来找出合适内存页。

1. 首先，它要判断根节点1是否已经被分配，如果是，则表示当前的个Chunk已经没有可用的空间，它需要去其的Chunk找了。
2. 接着，随机选择当前节点的某一个子节点（2或者3），判断是否有可用空间
3. 重复步骤2，直到找到可用的内存页

> 先写到这里，以下剩余内容的目录。另外，对`memoryMap`中的值，也需要进一步的说明,
> 因为关系具体的内存位置。
> 
> 写博客果然比较费事！

### 页内分配

# 内存的释放

## 内存页的释放

## 页内释放

# 其它

## ByteBuf实例缓存 


