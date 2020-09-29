---
layout: post
title: Netty设计思想之Reactor
description: "The Design of Netty - Reactor"
modified: 2019-08-01
tags: [Netty]
categories: [Netty]
---

作为流量的出入口，高性能的网络I/O服务对于提升系统吞吐量无疑是非常重要的。特别的，对于超大规模并发的网络应用程序来说，可扩展性（Scalability）是支撑高并发的必要条件。
可扩展性意味着随着系统资源的增加，性能的提升是连续的。所以需要尽可能的满足如下两个条件：

* 高吞吐，低延迟
* 较少的资源消耗

[事件驱动编程](https://en.wikipedia.org/wiki/Event-driven_programming)是一种由事件（Event）决定程序流程（Control flow）的编程范式。一般来说，会有一个主循环线程监听事件，当事件触发时指定相应的处理器（Handler）进行处理。这种范式广泛用在前端领域，针对UI事件做一系列不同的处理。

在网络编程领域中，也有一种事件驱动编程的具体实现，称为[Reactor模式](https://en.wikipedia.org/wiki/Reactor_pattern)。Reactor线程监听I/O事件，并在I/O事件准备就绪时，分发至具体的处理器进行处理。

Reactor模式相比传统的多线程IO处理，优势在于：

* 较少的资源消耗(线程、内存)来管理大量的连接
* 避免频繁的线程创建，减少了上下文切换

其高效的原因在于网络I/O是耗时操作，将等待I/O数据就绪的过程抽象成为事件监听并用单独的线程处理，而具体的处理逻辑可以由其他业务线程处理，这样就不会阻塞业务线程从而增大了系统的吞吐量。缺点是提升了编码的复杂度。

Java的NIO网络库封装了的非阻塞I/O的系统调用。Netty则基于Java NIO和Reactor模式设计了一套灵活的、可扩展的网络事件驱动框架。Netty的核心思想就是Reactor模式，并且屏蔽了编码的复杂度，提供了一套简洁的API快速高效的实现网络编程。

本文追本溯源，主要介绍下Netty的设计思想的背景知识。

## 网络I/O模型
对于网络程序来说，其主要的操作可以简单归纳为如下几步：

* 建立（Connect）连接
* 读取（Read）请求
* 解码（Decode）请求
* 处理请求（Process）请求
* 编码（Encode）响应
* 返回（Write）响应

由于网络I/O一般需要较长的处理时间，如果让进行系统调用的线程一直阻塞在网络I/O上会显著浪费CPU资源。所以网络I/O系统调用又分为阻塞I/O和非阻塞I/O，这边**阻塞**的意思是系统内核的数据是否准备好。非阻塞I/O的系统调用如下图所示：

{% capture images %}
    /images/nio.png
{% endcapture %}
{% include gallery images=images caption="" cols=2 %}

非阻塞I/O和阻塞I/O区别在于系统内核的数据在没有准备好的情况下是否会阻塞调用线程。非阻塞I/O是直接返回结果，而阻塞I/O则会阻塞调用线程。

如果需要管理多个连接的话，每个连接等待I/O事件的过程是一样的，可以用一个线程管理，称为I/O多路复用（I/O multiplexing），一次系统调用可以返回多个准备好的I/O事件。比如linux中的`select`或者`epoll`系统调用。I/O多路复用的系统调用如下图所示：

{% capture images %}
    /images/iom.png
{% endcapture %}
{% include gallery images=images caption="" cols=2 %}

我们可以发现I/O多路复用的系统调用其实就是事件驱动模型中的监听器，可以很自然的实现Reactor模式。

## Java NIO
Java NIO（Non-blocking I/O）网络库封装了I/O多路复用的系统调用，提供了I/O多路复用的能力。主要的接口实现有如下：

* Channels，支持非阻塞调用的socket或是文件
* Buffers, 可以直接读写的缓冲对象
* Selectors, 返回I/O事件的集合
* SelectorKeys, 维护I/O事件状态和绑定的数据

NIO的非阻塞的意思是业务线程可以以非阻塞的方式读取I/O数据。

## Reactor
Reactor模式可以有多种架构，最基础的是单Reactor线程架构，所有的I/O事件都由同一个线程循环监听分发。但为了更好的利用现代多核CPU，可以使用多Reactor线程架构，其架构如下图所示：

{% capture images %}
    /images/multi_reactor.png
{% endcapture %}
{% include gallery images=images caption="" cols=2 %}

主要模块如下：

* **Main Reactor**，负责连接事件的监听，并将连接事件分发至Acceptor
* **Acceptor**，连接事件处理器，负责接收新连接，并将连接信息注册至Sub Reactor
* **Sub Reactor**，负责读写事件的监听，并将读写事件分发至对应的处理器，可以新建多个Sub Reactor并发处理。
* **Write**，写事件处理器，将数据刷新到底层缓冲区
* **Read**，读事件处理器，可以将读取到的数据分发至业务线程池进行并发处理
* **Worker Thread**，业务线程池，可以进行编码、解码、数据处理等操作


## Netty实现
Netty关于I/O事件处理的设计思想其本质就是Reactor模式，采用的架构也是最先进的多Reactor线程架构。主要的实现类如下：

* **EventLoop**，事件循环线程，相当于Reactor线程，可以分别构建Main Reactor和Sub Reactor
* **EvenLoopGroup**，事件循环线程池，相当于Sub Reactor线程池，针对读写事件可以并发处理
* **ChannelHandler**，Channel事件处理器，负责I/O事件处理
* **EventExecutorGroup**，每个处理器可以指定其专有线程池，相当于上面的Worker Thread

我们可以看一下用Netty实现的一个简单的EchoServer

```java
// Configure the server.
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(8);
final EchoServerHandler serverHandler = new EchoServerHandler();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) throws Exception {
             ChannelPipeline p = ch.pipeline();
             p.addLast(serverHandler);
         }
     });

    // Start the server.
    ChannelFuture f = b.bind(PORT).sync();

    // Wait until the server socket is closed.
    f.channel().closeFuture().sync();
} finally {
    // Shut down all event loops to terminate all threads.
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

* `bossGroup`是初始化的Main Reactor线程，一般只用设置一个线程，因为接收新的连接没有额外的处理逻辑

* `workerGroup`是初始化的Sub Reactor线程池，处理读写事件，这边设置了8个线程，可以根据CPU核数设定

* `EchoServerHandler`是数据处理器，将客户端的数据原封不动的返回给客户端，这边没有为此Handler指定专有线程池，默认使用`workerGroup`线程池


## 总结
Netty作为高性能的基于事件驱动的网络框架，其事件驱动的本质就是多Reactor线程架构，并结合NIO的非阻塞特性实现了高性能的I/O事件处理。这对我们理解Netty的设计思想有非常大的作用。



