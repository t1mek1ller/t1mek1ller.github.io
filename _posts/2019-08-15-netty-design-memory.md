---
layout: post
title: Netty设计思想之内存模型
description: "The Design of Netty - Memory"
modified: 2019-08-01
tags: [Netty]
categories: [Netty]
---


Netty作为高性能网络库，其高性能主要归功于两个方面的设计：一方面是基于Reactor的事件驱动线程模型，全异步的数据处理提高的系统的吞吐量；另一方面，是精心设计和实现的内存模型，采用内存池，减少锁竞争，避免内存拷贝。

本文主要介绍下Netty的内存模型。


## 新的数据缓冲对象 - ByteBuf
为什么需要新的数据缓冲对象？不同于Java NIO的`Bytebuffer`，Netty设计的`Bytebuf`提供了更灵活的操作和更高级的功能，包括如下：

* 数据读写双指针，区别于`ByteBuffer`的单指针，`ByteBuf`不需要调用flip操作进行读写切换
* 零拷贝，针对组合类型的数据操作，可以用`CompositeByteBuf`避免多余的拷贝
* 池化，采用内存池避免频繁内存申请和释放，减少GC压力
* 引用计数，基于引用计数机制进行显示的内存释放
* 自动扩容，类似`StringBuilder`的自动扩容机制
* 链式调用，方便编程，兼具可读性


`ByteBuf`和`ByteBuffer`的基本结构对比如下：

{% capture images %}
    /images/netty_design_memory_bytebuf.png
{% endcapture %}
{% include gallery images=images caption="" cols=2 %}


由上图可见，`ByteBuf`的读指针`readerIndex`和写指针`writerIndex`分离，使得操作更加直观，而不用像`ByteBuffer`一样分为读模式和写模式，模式切换还需要调用`ByteBuffer.flip()`操作进行切换，增加了编程的复杂度、易出错。

另外，`ByteBuffer`的容量初始化后是不可变的，而`ByteBuf`可以动态扩容。


[零拷贝（zero copy）](https://zh.wikipedia.org/wiki/%E9%9B%B6%E5%A4%8D%E5%88%B6)一般是指操作系统层面避免内核态到用户态的冗余数据拷贝，直接在内核态进行数据复制。Netty中的零拷贝主要是针对用户态中的部分场景尽量避免数据的冗余复制。比如组合多个缓冲可以使用`CompositeByteBuf`对象避免额外的内存拷贝。


池化是手动管理申请的内存，避免频繁的内存申请和释放，减少GC压力。虽然这和JVM的GC设计理念背道而驰，但是针对堆外内存的频繁申请和释放会影响性能，对于延迟敏感的网络应用程序来说是不可接受的。Netty设计了`PooledByteBuf`屏蔽了底层具体数据缓冲的实现（无论是Heap Buffer还是Direct Buffer），通过统一的接口（`PooledByteBufferAllocator`）进行内存的分配，并基于引用计数的机制回收对象。这种设计带来的负面影响是开发人员需要手动释放对象，但是对于其带来的收益，这点成本可以接受。

## 内存模型 - PoolThreadCache
Netty的内存池实现参考了[jemelloc](http://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf)以及[facebook的jemalloc实现](https://www.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/480222803919)。其主要思想如下：

* 小内存分配基于size等级，并且遵循低地址优先原则，可以减少内存碎片
* 精心选择size等级数目和间隔。如果间隔过大，对象会无法利用尾部空间（内部碎片）。如果size等级数目过多，又会造成很多内存浪费（外部碎片）
* 严格限制分配程序元数据的开销。一般小于总内存的2%
* 减少活动页面数目。操作系统内核是以页为单位管理虚拟内存，尽量将数据聚集在少数几页，可以减少页面切换
* 最小化锁竞争。采用独立的arena机制和线程本地cache，来减少线程之间的同步消耗

我们先看下Netty内存管理的全局视图：


{% capture images %}
    /images/netty_design_memory_birdview.png
{% endcapture %}
{% include gallery images=images caption="" cols=2 %}

主要结构分为如下几个部分：

* **PoolThreadCache**, 线程本地缓存，缓存之前申请过的内存对象，初始化时会选择利用率最少的PoolArena对象，基本上每个线程可以拥有自己的PoolArena对象以避免锁竞争。
* **PoolArena**，全局内存分配器，定义了小内存分配的size等级及间隔、一般内存的分配流程，以及超大内存的分配，其主要元素有：
    * **tinySubpagePools**，小内存分配数组，tiny是指小于512B的内存分配，每个数组是`PooSubpage`组成的链表，数组间大小间隔为16B，，所以数组共有32个链表
    * **smallSubpagePools**，小内存分配数组，small是指小于一个page大小（默认8k）的内存分配，每个数组也是由`PooSubpage`组成的链表，数组长度为log(pageSize/512)-1
    * **PoolChunkList**，PoolChunk的列表，可以根据利用率划分多个列表，在内存分配中动态改变
* **PoolChunk**，负责一般内存大小（大于page）的分配。page是内存管理的最小单元，默认8k。chunk是page为单元的集合，默认16M。基于完全平衡二叉树管理chunk，树中的节点是一个page，分配的时候会找到合适层进行page分配。每一层page大小的总和等于chunk大小
* **PoolSubpage**，负责小内存分配（小于page），是从chunk的叶子节点中分配得到。基于bitmap划分page。

