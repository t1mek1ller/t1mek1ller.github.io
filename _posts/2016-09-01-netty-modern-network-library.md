---
layout: post
title: Netty高性能网络库
description: "netty in practice"
modified: 2016-09-01
tags: [Netty]
categories: [Netty, Network]
---

现代分布式系统中的各个节点之间都需要通信，进行数据交换、状态同步等。高性能的网络通信组件是分布式系统的实现基础，由于TCP/IP协议细节实在太多，如果从零开始短时间内实现一个高性能的网络库，实属不易，成本也过高。本着拿来主义的原则，向开源世界中汲取精华总是不坏的，毕竟社会是分工合作的。[Netty](http://netty.io/)则是一个优秀的开源网络库，初衷是简单快速的实现高性能的网络层通讯。自己在生产实践中使用过Netty实现私有协议进行系统间通信，开发效率大大提高，优雅的api设计也使得开发过程非常舒适，采用的异步事件驱动模型(IO多路复用)让其性能非常优秀。下面简单介绍一下Netty的设计和使用。

## 问题定义
面临的问题是如何简单快速的实现高性能的网络通信？

现有的http客户端和服务器可以解决大部分的网络通信需求，但是有的场景下我们需要更精简的网络通讯协议，比如实时性要求非常高的IM即时通信、消息中间件、RPC等。这种情况下需要实现私有协议，个性化自己的通信需求，使得通信方式更加简洁高效。除了私有协议，还有其他公共协议比如SSL/TSL、Google protobuf、WebSocket等都需要支持。一个通用的网络框架，不仅仅是**高性能**，更重要的是让使用者**简单快速**的实现自己的私有协议。

## Netty特点
Netty采用异步事件驱动模型，可以快速开发出高性能、稳定可扩展的网络通信服务。

设计上：

* 对于阻塞/非阻塞的通信方式，有统一的API
* 基于灵活可扩展的事件驱动模型可以清晰的分离关注点
* 高度可配置的线程模型，SEDA(Staged Event-Driven Architecture，核心思想是把一个请求处理过程分成几个Stage，不同资源消耗的Stage使用不同数量的线程来处理，Stage间使用事件驱动的异步通信模式。)
* 真正的无连接的数据报通信支持（这个不太理解...）

使用简单：

* 良好的文档、用户指南、使用例子
* 没有第三方依赖

高性能：

* 高吞吐、低延迟
* 底资源消耗
* 最小化不必要的内存拷贝（netty实现的buteBuf数据结构，优化了数据拷贝）

安全性：

* 完整的SSL/TLS支持

[Netty的各个组件](https://rocketmq.apache.org/assets/images/rmq-model.png)如下图所示：

{% capture images %}
	/images/netty-components.png
{% endcapture %}
{% include gallery images=images caption="Netty组件" cols=1 %}

对于netty与其他网络库的区别，官网给出的回答很有意思：

> The answer is the philosophy it is built on. Netty is designed to give you the most comfortable experience both in terms of the API and the implementation from the day one. It is not something tangible but you will realize that this philosophy will make your life much easier as you read this guide and play with Netty.

简短的说，就是人生苦短，请用netty。


## Netty使用要点

### 管道流模型ChannelPipeline
Netty是基于管道流模型处理IO请求和响应的。对于每一个channel会创建一个对应ChannelPipeline，这意味是线程安全的，不同的channel之间不会相互影响，所以可以动态的增加和删除Pipeline中的处理器。

[ChannelPipeline的事件流](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)如下图所示：

{% capture images %}
	/images/netty-channelpipeline.png
{% endcapture %}
{% include gallery images=images caption="ChannelPipeline事件流" cols=1 %}

Pipeline中的处理器（ChannelHandler）分为两种，Inbound表示从socket中读数据后自底向上的处理过程，Outbound表示往socket写数据前自顶向下的处理过程。通过这种方式，可以灵活配置数据处理器。

一个典型的使用netty的服务端例子：

{% highlight Java %}
		
		// 启动server端服务的辅助类
		ServerBootstrap serverBootstrap;

		// selector线程池
    	EventLoopGroup eventLoopGroupSelector;

    	// boss线程池
    	EventLoopGroup eventLoopGroupBoss;

    	// 应用配置（非netty类）
    	NettyServerConfig nettyServerConfig;

		// boss线程池
        eventLoopGroupBoss = new NioEventLoopGroup(1, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, String.format("NettyBoss_%d", this.threadIndex.incrementAndGet()));
            }
        });

        // linux使用epoll性能更佳
        if (useEpoll()) {
             eventLoopGroupSelector = new EpollEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerEPOLLSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        } else {
            eventLoopGroupSelector = new NioEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerNIOSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        }

        // 事件处理线程池
        DefaultEventExecutorGroup defaultEventExecutorGroup = new DefaultEventExecutorGroup(
            nettyServerConfig.getServerWorkerThreads(),
            new ThreadFactory() {

                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
                }
            });

        ServerBootstrap childHandler =
            serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
                .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .option(ChannelOption.SO_REUSEADDR, true)
                // 关闭TCP自身的保活机制，由应用层进行心跳逻辑
                .option(ChannelOption.SO_KEEPALIVE, false)
                .childOption(ChannelOption.TCP_NODELAY, true)
                // 发送和接收缓冲区
                .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
                .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
                .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                            .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME,
                            	// SSL/TLS处理
                                new HandshakeHandler(TlsSystemConfig.tlsMode))
                            .addLast(defaultEventExecutorGroup,
                            	// 私有协议编解码
                                new NettyEncoder(),
                                new NettyDecoder(),
                                // 空闲状态处理
                                new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                                new NettyConnectManageHandler(),
                                new NettyServerHandler()
                            );
                    }
                });

        // Netty默认不使用内存池，需要在创建客户端或者服务端的时候进行指定
        // 由于ByteBuf使用的是堆外内存，不受jvm控制，所以内存的申请和释放必须成对出现，即retain()和release()要成对出现，否则会导致内存泄露。
        if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
            childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
        }

        try {
            ChannelFuture sync = this.serverBootstrap.bind().sync();
        } catch (InterruptedException e1) {
            throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
        }
{% endhighlight %}



### 心跳处理
如果是长连接服务，需要感知对方的状态，激活或关闭空闲的连接，一般是通过发送心跳包的方式进行实现。TCP本身有KEPP-ALIVE机制，默认是7200秒发送探活包，这个时间明显太长，为了更好的控制的心跳机制，应用层会自己实现心跳机制。

Netty提供了IdleStateHandler进行空闲链路的管理，可以监测到读空闲、写空闲、读写空闲的事件，可以在这些事件触发时做一些处理。需要注意的是不应该在这些handler里面执行繁重任务，否则会阻塞IO线程。空闲时间可以设置为120s或180s，应用层定期（比如30s）发送心跳包。

### 引用计数对象ByteBuf
从Netty4开始，使用堆外内存实现零拷贝，避免GC。但是申请的内存就需要自己来管理，可以通过引用计数这一方式判断对象是否可以回收。真正使用的时候需要注意引用计数的处理，否则会造成内存泄漏。netty也[提供了泄漏检测的方式](http://netty.io/wiki/reference-counted-objects.html#wiki-h2-10)，通过给jvm选项上添加**-Dio.netty.leakDetectionLevel=advanced**可以打印出泄漏代码。



