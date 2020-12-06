---
layout: post
title: Netty设计思想之API
description: "The Design of Netty - API"
modified: 2019-08-01
tags: [Netty]
categories: [Netty]
---


Netty作为十分流行的NIO网络库，除了高性能的优点外，另一大优势就是其简洁易用的API，使得开发者们可以快速高效的开发网络应用程序。这得益于Netty作者多年的网络协议实现经验，让网络库变得简单、易用、稳定、高性能和可扩展。

网络库那么多，为什么要使用Netty呢？官方的回答很有意思：

> The answer is the philosophy it is built on. Netty is designed to give you the most comfortable experience both in terms of the API and the implementation from the day one. It is not something tangible but you will realize that this philosophy will make your life much easier as you read this guide and play with Netty.

**舒适的使用体验**是Netty的设计哲学。简而言之，**人生苦短，请用Netty**。

API作为软件系统的门面，是软件系统思维模型的直接体现，其设计的合理性决定着软件系统后续的可维护性。[好的API的设计一般遵循如下几个规则](https://mp.weixin.qq.com/s/qWrSyzJ54YEw8sLCxAEKlA)：

* **提供清晰的思维模型**，设计良好的API让维护者和使用者能够很容易理解到设计时要传达的模型。带来理解、调试、测试、代码扩展和系统维护性的提升。
* **简单**，尽可能的设计简单，过于复杂的设计会让使用者一头雾水，后续可维护性也会大大下降。
* **容许多个实现**，API的设计不应加入实现细节，方便后续扩展。

本文从API的角度看一下Netty的设计，从其API中可以知晓其背后的思维模型。

## Boostrap
我们首先从开发者的角度，看一下如何用Netty快速实现启动一个网络应用。`AbstractBootstrap`是Netty提供的以链式的方式快速配置并启动的脚手架抽象类，提供了`Bootstrap`和`ServerBootstrap`分别对应client和server的配置启动。其API和继承类图如下所示（这里主要列举核心的API）：


{% capture images %}
    /images/netty_design_api_bootstrap.png
{% endcapture %}
{% include gallery images=images caption="" cols=2 %}

`AbstractBootstrap`的API主要分为四类：

* **数据通道（Channel）**，提供配置底层Socket数据通道类型、通道配置和自定义通道属性的功能（这里`Channel`接口是Netty自定义的通道接口，区别JDK中的`Channel`，提供了更加丰富的操作）。

* **事件循环（Reactor）线程池**，提供配置Reactor线程池（Reactor模式可以参见[Netty设计思想之Reactor](https://t1mek1ller.github.io/2019/08/01/netty-design-reactor/)）的功能，针对server端可以配置多Reactor线程模型。


* **事件处理器**，提供配置`Channel`的事件处理器的功能，可以通过`ChannelPipeline`的方式链式处理，每个处理器可以单独指定线程池。

* **启动**，提供配置地址、端口及启动程序的功能。

核心API的描述如下表：

{% capture images %}
    /images/netty_design_api_bootstrap_table.png
{% endcapture %}
{% include gallery images=images caption="" cols=1 %}

从上面的API的设计，我们其实可以知道Netty两个核心模型：

* [多Reactor线程模型](https://en.wikipedia.org/wiki/Reactor_pattern)，I/O事件驱动的线程模型，连接事件和读写事件分离分别监听及处理
* [SEDA（staged event-driven architecture，分阶段事件驱动架构)](https://en.wikipedia.org/wiki/Staged_event-driven_architecture)，将整个处理流程分为多个阶段，每个阶段的事件处理器可以单独指定线程池。

上面两个模型后面会详细介绍，我们以server端为例看下Netty的实现方式：

```java
// 配置连接事件的reactor线程
EventLoopGroup bossGroup = new NioEventLoopGroup(1);

// 配置读写事件的reactor线程
EventLoopGroup workerGroup = new NioEventLoopGroup(8);

// 处理器
final EchoServerHandler serverHandler = new EchoServerHandler();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
      // 指定ServerSocketChannel具体实现
     .channel(NioServerSocketChannel.class)

     // 指定新建立的连接的链式处理器
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) throws Exception {
             ChannelPipeline p = ch.pipeline();
             p.addLast(serverHandler);
         }
     });

    // 绑定地址、端口并启动
    ChannelFuture f = b.bind(PORT).sync();

    // 等待直到ServerSocket关闭
    f.channel().closeFuture().sync();
} finally {
    // 关闭reactor线程池
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```


## Channel, ChannelFuture
`Channel`作为Netty的核心组件之一，是对Socket数据通道的抽象，负责I/O操作，比如`read`、`write`、`connect`、`bind`。I/O操作的具体实现是由底层的网络库提供的，比如NIO、Epoll等。

`Channel`的接口设计如下图所示：
{% capture images %}
    /images/netty_design_api_channel.png
{% endcapture %}
{% include gallery images=images caption="" cols=1 %}

从`Channel`的API设计可以看出，`Channel`主要职责为:
* `Channel`的当前状态，比如是否连接、是否活跃、是否可写等等。
* `Channel`的网络通信配置，比如读写缓冲。
* `Channel`支持的I/O操作，具体的I/O操作由内部`Unsafe`接口实现。
* `Channel`注册的`EventLoop`, 一个`Channel`只可以注册到一个`EventLoop`中，一个`EventLoop`可以管理多个`Channel`
* `Channel`的数据处理由其关联的`ChannelPipeline`负责。

Netty作为异步的NIO框架，其所有的I/O操作都是异步的。这意味着对I/O操作的调用是立马返回的，Netty提供了`ChannelFuture`接口在操作完成后回调`listener`通知结果。

`ChannelFuture`的接口设计如下图所示：
{% capture images %}
    /images/netty_design_api_channelfuture.png
{% endcapture %}
{% include gallery images=images caption="" cols=1 %}

`ChannelFuture`主要是在`Future`的基础上添加了获取相应`Channel`的方法，以及设置`listener`回调的接口，在I/O操作完成时可以获取到操作结果。

这里需要注意的是在`listener`回调函数中不宜使用长时间的阻塞操作，因为I/O操作一般都是在I/O线程中执行的，长时间的阻塞操作会阻塞I/O线程从而不能响应新的I/O事件。

## ChannelHandler, ChanelPipeline, ChannelHandlerContext
`ChannelPipeline`是Netty的数据处理组件，由一系列`ChannelHandler`组成，以管道流的方式处理Channel相关的上行和下行数据流。这种设计解耦了网络层和数据处理层，使得开发者只需要实现、复用、组装自己的`ChannelHandler`，而不用关心底层的网络实现。

整个`ChannelPipeline`的接口设计如下图所示：
{% capture images %}
    /images/netty_design_api_channelpipeline_class.png
{% endcapture %}
{% include gallery images=images caption="" cols=1 %}

Netty将I/O操作分为上行（`ChannelInboundInvoker`）和下行（`ChannelOutboundInvoker`）两部分。相应的，事件处理器也分为上行处理器（`ChannelInboundHandler`）和下行处理器（`ChannelOutboundHandler`）。

`ChannelPipeline`并不是直接操作`ChannelHandler`，而是通过`ChannelHandlerContext`进行交互，`ChannelHandlerContext`的作用在于：

* **事件通知**，通过保存上下文处理器，可以实现事件的链式处理
* **动态修改**，可以在运行时动态添加、删除、替换`ChannelHandler`
* **状态信息保存**，如果一个`ChannelHandler`是有状态的（含有状态信息，比如以成员变量的形式），那就必须在为`Channel`初始化`ChannelPipeline`时创建`ChannelHandler`实例。为了避免过多的实例创建，可以选择将状态信息保存在`ChannelHandlerContext`中，这样不同的`ChannelPipeline`可以就复用同一个实例。这种可以被复用的`ChannelHandler`，可以由`@Sharable`注解标识。所以一个`ChannelHandler`可能被多个`ChannelHandlerContext`引用。

Netty的数据流由下图所示：

{% capture images %}
    /images/netty_design_api_channelpipeline.png
{% endcapture %}
{% include gallery images=images caption="" cols=1 %}

网络层socket读取的数据会按配置顺序自低向上的由各个上行处理器（`ChannelInboundHandler`）处理。同样的，应用层写入的数据也会按配置顺序自顶向下的由个下行处理器（`ChannelOutboundHandler`）处理，最后写入到网络层的socket中。相邻`ChannelHandler`之间是由`ChannelHandlerContext`基于事件机制触发的。

我们注意到`ChannelPipeline`在添加`ChannelHandler`时有如下两个接口：

```java
    /**
     * Appends a {@link ChannelHandler} at the last position of this pipeline.
     *
     * @param name     the name of the handler to append
     * @param handler  the handler to append
     *
     * @throws IllegalArgumentException
     *         if there's an entry with the same name already in the pipeline
     * @throws NullPointerException
     *         if the specified handler is {@code null}
     */
    ChannelPipeline addLast(String name, ChannelHandler handler);

    /**
     * Appends a {@link ChannelHandler} at the last position of this pipeline.
     *
     * @param group    the {@link EventExecutorGroup} which will be used to execute the {@link ChannelHandler}
     *                 methods
     * @param name     the name of the handler to append
     * @param handler  the handler to append
     *
     * @throws IllegalArgumentException
     *         if there's an entry with the same name already in the pipeline
     * @throws NullPointerException
     *         if the specified handler is {@code null}
     */
    ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler);
```

这两个接口的不同之处在于，第二个接口指定了`ChannelHandler`执行的线程池`EventExecutorGroup`。默认情况下，`ChannelHandler`的处理操作是在I/O线程中执行的，但是如果`ChannelHandler`中有耗时的阻塞操作，就需要指定专有线程池以免阻塞I/O线程。

`ChannelPipeline`这种设计可以称之为[SEDA（staged event-driven architecture，分阶段事件驱动架构)](https://en.wikipedia.org/wiki/Staged_event-driven_architecture)，将数据流的处理划分为不同的阶段，阶段之间通过事件的方式异步通知，每个阶段可以有单独的线程池进行处理。这种架构的好处在于，将数据处理异步化、模块化，提高吞吐量的同时简化编程。


## EventLoop, EventLoopGroup
`EventLoop`是Netty提供的Reactor线程组件，主要处理网络I/O事件及并发任务。相对应的，`EventLoopGroup`则是Reactor线程池组件。Netty线程模型就是多Reactor线程模型，参见[Netty设计思想之Reactor](https://t1mek1ller.github.io/2019/08/01/netty-design-reactor/)。

`EventLoop`的API类图如下所示（引用[Netty In Action](https://www.manning.com/books/netty-in-action)）：

{% capture images %}
    /images/netty_design_api_eventloop.png
{% endcapture %}
{% include gallery images=images caption="" cols=1 %}

由上图可知，`EventLoop`是直接扩展自`java.util.concurrent.ScheduledExecutorService`进行任务处理，以及`io.netty.util.concurrent.EventExecutor`进行线程调度，并添加一些`Channel`相关接口进行I/O操作。

一个`EventLoop`对应一个`Thread`，上面也说到，一个`EventLoop`可以服务（注册）多个`Channel`，这得益于NIO的`Selector`接口进行I/O多路复用。

注意到`EventLoop`有如下接口：
```java
    /**
     * Calls {@link #inEventLoop(Thread)} with {@link Thread#currentThread()} as argument
     */
    boolean inEventLoop();
```

这个接口揭示了`EventLoop`的执行逻辑：任务的执行必须在本线程中执行，否则会添加至任务队列之中。

`EventLoopGroup`是由多个`EventLoop`组成的，注册`Channel`时会唯一分配一个`EventLoop`，`Channel`后续相关的I/O事件都由这个`EventLoop`处理，事件和任务都以FIFO的顺序执行，可以避免额外的线程同步操作，保证了高性能。


## 总结
从Netty的API设计之中，我们可以看到其两个核心模型：多Reactor线程模型和SEDA线程模型，前者让Netty保持了高性能的特性，后者让Netty易于使用和维护。

还是那句话，人生苦短，请用Netty。


