---
layout: post
title: Reactor Netty的回压设计
description: "Backpressure in reactor-netty"
modified: 2020-06-01
tags: [Reactor]
categories: [Netty, Reactor Netty, Reactor, Reactive Programming]
---

[Reactor Netty](https://github.com/reactor/reactor-netty)是基于[Reactor Core](https://github.com/reactor/reactor-core)和[Netty](https://github.com/netty/netty)构建的反应式网络库。通过在Netty上构建一层反应式层，引入回压机制，使得系统具有即时响应性（responsive）、弹性（elastic）和回弹性（resilient），这也是[反应式宣言](https://www.reactivemanifesto.org/) 所倡导的反应式架构应具有的特性。Reactor Netty也是[Spring WebFlux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux)反应式的默认底层实现。

{% capture images %}
    /images/reactor_netty_arch.png
{% endcapture %}
{% include gallery images=images cols=1 %}


本文深入其反应式层，看看是如何实现所谓的回压特性。


## Echo Server
先看一个官方的[EchoServer](https://github.com/reactor/reactor-netty/blob/master/reactor-netty-examples/src/main/java/reactor/netty/examples/tcp/echo/EchoServer.java)的代码实现，这里传输层采用TCP，本文主要探讨传输层基于TCP之上的反应式层是如何实现的，HTTP和UDP的实现原理大同小异。

```java
/**
 * A TCP server that sends back the received content.
 *
 * @author Violeta Georgieva
 */
public final class EchoServer {

    static final boolean SECURE = System.getProperty("secure") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", SECURE ? "8443" : "8080"));
    static final boolean WIRETAP = System.getProperty("wiretap") != null;

    public static void main(String[] args) throws Exception {
        TcpServer server =
                TcpServer.create()
                         .port(PORT)
                         .wiretap(WIRETAP)
                         .handle((in, out) -> out.send(in.receive().retain()));

        if (SECURE) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            server = server.secure(
                    spec -> spec.sslContext(SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey())));
        }

        server.bindNow()
              .onDispose()
              .block();
    }
}
```

使用体验上和反应式编程保持一致，均是惰性链式调用，先进行装配（assembly），最后通过block函数启动整个流程。

`TcpServer`是Reactor Netty提供的配置和启动一个TCP服务器的类。它隐藏了大部分Netty的ServerBootstrap的配置方法，重新对外提供了一套配置接口，虽然功能上并无差别，但内部实现了反应式流的语义。

除了一般的port、ssl配置，需要额外注意的是`TcpServer.handle`方法，这个给应用层设置数据读写操作的接口，下面会详细介绍。

另外，TcpServer进行bind操作后会返回`DisposableServer`，封装了`ServerSocketChannel`，可以通过`block`订阅channel的close事件。


## 反应式层架构

在继续探讨Reactor Netty反应式层实现之前，先祭出一张整体架构图，可以让我们对其实现有个直观的感受。

{% capture images %}
    /images/reactor_netty_stream.png
{% endcapture %}
{% include gallery images=images%}

Reactor Netty的反应式层主要由两大块构建：

* Handler，基于原生的netty handler的实现，包括三类handler
    * Reactor Netty Handler，库自己实现的handler，优先级最高，前置于用户自定义handler，比如`reactor.left.accessLogHandler`, `reactor.left.channelMetricsHandler`等。可以在`NettyPipeline`接口中找到目前实现的handler。
    * 用户自定义handler，在装配期间，用户可以指定自定义handler，做一些额外操作，比如codec相关的操作。
    * Reactive Bridge Handler, 这个是较为特殊的handler，作为netty和reactive的桥接器，具体实现类是`ChannelOperationsHandler`

* Reactive，反应式功能的实现，包括四类模块
    * ChannelOperations，实现了`NettyInBound`和`NettyOutBound`接口，负责发布与订阅数据，与channel的生命周期一致。它会接收数据并传递至`FluxReceiver`，而且由它来订阅经由用户处理完毕待发送出去的数据。
    * FluxReceive，扩展自Flux，从ChannelOperations接受数据，确保数据的发布都在channel的eventloop线程中，并实现了回压机制，是数据流的源头。
    * MonoSendMany，负责接收用户处理完毕待发送的数据，作为FluxReceive的下游。
    * 用户自定义I/O handler, 采用反应式编程进行数据的处理和发送，无需关心底层的实现。


## Reactive Bridge Handler
Reactive Bridge Handler作为netty和reactive的桥接层，是数据流的入口并做一些准备工作，`ChannelOperationsHandler`是其实现类。主要实现了三个功能：

* 连接建立的时候创建`ChannelOperations`，绑定到对应的channel中，装配好所有的反应式流并订阅。
* 读取channel数据并交给`ChannelOperations`处理
* 连接关闭的时候触发反应式流的取消


## To be continued......


## 参考文献
1. [Reactor Netty](https://projectreactor.io/docs/netty/release/reference/index.html#getting-started-introducing-reactor-netty)
2. [Reactor Core](https://github.com/reactor/reactor-core)
3. [Netty](https://github.com/netty/netty)
4. [The Reactive Manifesto](https://www.reactivemanifesto.org/)
5. [EchoServer](https://github.com/reactor/reactor-netty/blob/master/reactor-netty-examples/src/main/java/reactor/netty/examples/tcp/echo/EchoServer.java)
6. [Spring WebFlux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux)




