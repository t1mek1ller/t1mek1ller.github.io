---
layout: post
title: Reactor Netty的回压设计
description: "Backpressure in reactor-netty"
modified: 2020-12-06
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
                    spec -> spec.sslContext(SslContextBuilder
                                                .forServer(ssc.certificate(), ssc.privateKey())));
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

* Handler，基于原生的Netty Handler的实现，包括三类Handler
    * Reactor Netty Handler，库自己实现的handler，优先级最高，前置于用户自定义handler，比如`reactor.left.accessLogHandler`, `reactor.left.channelMetricsHandler`等。可以在`NettyPipeline`接口中找到目前实现的handler。
    * 用户自定义handler，在装配期间，用户可以指定自定义handler，做一些额外操作，比如codec相关的操作。
    * Reactive Bridge Handler, 这个是较为特殊的handler，作为netty和reactive的桥接器，具体实现类是`ChannelOperationsHandler`

* Reactive，反应式功能的实现，包括四类模块
    * ChannelOperations，实现了`NettyInBound`和`NettyOutBound`接口，负责发布与订阅数据，与channel的生命周期一致。它会接收数据并传递至`FluxReceiver`，而且由它来订阅经由用户处理完毕待发送出去的数据。
    * FluxReceive，扩展自Flux，从ChannelOperations接受数据，确保数据的发布都在channel的eventloop线程中，并实现了回压机制，是数据流的源头。
    * MonoSendMany，负责接收用户处理完毕待发送的数据，作为FluxReceive的下游。
    * 用户自定义I/O handler, 采用反应式编程进行数据的处理和发送，无需关心底层的实现。


## 一些重要组件

### Reactive Bridge Handler
Reactive Bridge Handler作为netty和reactive的桥接层。它作为整个数据流的入口，为反应式流的装配和订阅做好了准备工作。`ChannelOperationsHandler`是其实现类，主要实现了三个功能：

* 连接建立的时候创建`ChannelOperations`，绑定到对应的channel中，装配好所有的反应式流并订阅。
* 读取channel数据并交给`ChannelOperations`处理
* 连接关闭的时候取消反应式流的订阅


### Connection
Connection是对连接的抽象接口，提供了channel的上下文信息以及读写相关操作。它会在连接建立的时候存储在channel的属性之中提供给其他组件进行channel相关操作。比如用户在注册netty pipeline handler的时候，会将他们放在正确的位置（如上述架构图所示）。


### NettyInbound和NettyOutbound
NettyInbound和NettyOutbound是开放给用户的读写接口。在`TcpServer.handle`的函数签名中，我们可以得知用户可以通过这个接口添加额外的操作并返回一个`Publisher`给Reactor Netty，由Reactor Netty负责进行订阅从而启动一个反应式流。

```java
public TcpServer handle(BiFunction<? super NettyInbound, 
                                   ? super NettyOutbound, 
                                   ? extends Publisher<Void>> handler)
```
相当于用户在流中间加上自定义的操作。


NettyInbound定义了接收channel数据的接口集，包括三个接口：
```java
public interface NettyInbound {


    /**
     * 获取ByteBuf流，可以给用户进一步的解码操作
     *
     * @return ByteBufFlux是Reactor Netty新增的Flux运算符，用于对ByteBuf做一些操作
     */
    ByteBufFlux receive()

    /**
     * 获取已经解码好的对象流
     *
     * @return 对象流Flux
     */
    Flux<?> receiveObject();

    /**
     * 注册Connection的回调
     *
     * @param withConnection connection callback
     *
     * @return the {@link Connection}
     */
    NettyInbound withConnection(Consumer<? super Connection> withConnection);
}

```

而NettyOutbound则定义了发送数据的接口集，部分需要实现的有如下接口：

```java
public interface NettyOutbound extends Publisher<Void> {

    /**
     * ByteBufAllocator，一般是采用对应channel的ByteBufAllocator
     *
     * @return the {@link ByteBufAllocator}
     */
    ByteBufAllocator alloc();

    /**
     * 发送ByteBuf数据给对端，会监听写入失败或连接中断等异常错误
     * @param dataStream 需要写出的数据流
     * @param predicate 是否需要立即进行flush操作
     *
     * @return A new {@link NettyOutbound} to append further send. It will emit a complete
     * signal successful sequence write (e.g. after "flush") or any error during write.
     */
    NettyOutbound send(Publisher<? extends ByteBuf> dataStream, Predicate<ByteBuf> predicate);



    /**
     * 发送对象给对端
     *
     * @param dataStream 需要写出的数据流
     * @param predicate 是否需要立即进行flush操作
     *
     * @return A Publisher to signal successful sequence write (e.g. after "flush") or any
     * error during write
     */
    NettyOutbound sendObject(Publisher<?> dataStream, Predicate<Object> predicate);

    /**
     * 发送对象给对端，会监听写入失败或连接中断等异常错误
     *
     * @param message 需要写出的对象
     *
     * @return A {@link Mono} to signal successful sequence write (e.g. after "flush") or
     * any error during write
     */
    NettyOutbound sendObject(Object message);


    /**
     * 支持初始化/清理操作的发送，可以用于发送文件
     *
     * @param sourceInput 状态生成器
     * @param mappedInput 待发送对象
     * @param sourceCleanup 状态清理
     * @param <S> 状态类型
     *
     * @return a new {@link NettyOutbound}
     */
    <S> NettyOutbound sendUsing(Callable<? extends S> sourceInput,
            BiFunction<? super Connection, ? super S, ?> mappedInput,
            Consumer<? super S> sourceCleanup);


    /**
     * 注册Connection回调
     *
     * @param withConnection connection callback
     *
     * @return the {@link Connection}
     */
    NettyOutbound withConnection(Consumer<? super Connection> withConnection);   

```


### ChannelOperations
ChannelOperations集成了针对channel的反应式相关操作，实现了`NettyInBound`和`NettyOutBound`接口，同样也实现了`Connection`接口，会在连接建立是初始化并绑定到channel中，最终用户使用的就是ChannelOperations。

另外，ChannelOperations还是整个反应式流的实际订阅者，在连接绑定完成之后启动反应式流，会监听整个流的complete和error事件并做对应的处理。


### ConnectionObserver
ConnectionObserver是Connection生命周期的事件监听器，针对连接的各个状态进行对应的处理。

```java
    /**
     * 针对Connection状态变化进行响应
     *
     * @param connection the connection reference
     * @param newState the new State
     */
    void onStateChange(Connection connection, State newState);

```

`TcpServer`提供了`doOnConnection`接口进行注册回调，当有新客户端连接时会进行通知。


## 反应式流的订阅时机
现在，我们可以来看一下针对连接的反应式流是在什么时机进行订阅的。对于每一个新建立的连接，都有一个反应式流与之相对应，反应式流的装配和订阅都是在连接建立的时候进行的。

我们可以从`ChannelOperationsHandler.channelActive`函数中看到处理过程：


```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) {

        if (ctx.channel().isActive()) {

            // Connection是对channel的包装，这边返回的是一个简单的SimpleConnection
            Connection c = Connection.from(ctx.channel());
            // listener就是ConnectionObserver，进行事件触发
            listener.onStateChange(c, ConnectionObserver.State.CONNECTED);

            // 创建ChannelOperations并绑定
            ChannelOperations<?, ?> ops = opsFactory.create(c, listener, null);
            if (ops != null) {
                // 绑定到channel的属性之中
                ops.bind();
                // 进行反应式流的订阅
                listener.onStateChange(ops, ConnectionObserver.State.CONFIGURED);
            }
        }
    }

```
Reactor Netty提供了一个默认的ConnectionServer对状态进行处理，称之为`ServerTransportDoOnConnection`，其实现如下：

```java
    static final class ServerTransportDoOnConnection implements ConnectionObserver {

        final ChannelGroup                 channelGroup;
        final Consumer<? super Connection> doOnConnection;

        ServerTransportDoOnConnection(@Nullable ChannelGroup channelGroup, 
            @Nullable Consumer<? super Connection> doOnConnection) {
            this.channelGroup = channelGroup;
            this.doOnConnection = doOnConnection;
        }

        @Override
        @SuppressWarnings("FutureReturnValueIgnored")
        public void onStateChange(Connection connection, State newState) {
            if (channelGroup != null && newState == State.CONNECTED) {
                channelGroup.add(connection.channel());
                return;
            }
            if (doOnConnection != null && newState == State.CONFIGURED) {
                try {
                    doOnConnection.accept(connection);
                }
                catch (Throwable t) {
                    log.error(format(connection.channel(), ""), t);
                    //"FutureReturnValueIgnored" this is deliberate
                    connection.channel().close();
                }
            }
        }
    }
```
从实现可以得知，会直接调用配置好的`doOnConnection`函数。那配置好的`doOnConnection`又是什么呢？

上面提到`TcpServer.handle`会注册用户自定义的处理函数，我们可以看下内部实现：

```java
    /**
     * Attaches an I/O handler to react on a connected client
     *
     * @param handler an I/O handler that can dispose underlying connection when
     * {@link Publisher} terminates.
     *
     * @return a new {@link TcpServer}
     */
    public TcpServer handle(BiFunction<? super NettyInbound, 
                                       ? super NettyOutbound, 
                                       ? extends Publisher<Void>> handler) {
        Objects.requireNonNull(handler, "handler");
        // 将用户的handler注册到ConnectionObserver中
        return doOnConnection(new OnConnectionHandle(handler));
    }
```
这边的关键是`OnConnectionHandle`，是对用户自定义handler的包装，实现如下：

```java

    static final class OnConnectionHandle implements Consumer<Connection> {

        final BiFunction<? super NettyInbound, 
        ? super NettyOutbound, 
        ? extends Publisher<Void>> handler;

        OnConnectionHandle(BiFunction<? super NettyInbound, 
            ? super NettyOutbound, 
            ? extends Publisher<Void>> handler) {
            this.handler = handler;
        }

        // 这里的Connection实际就是ChannelOperations
        @Override
        public void accept(Connection c) {
            if (log.isDebugEnabled()) {
                log.debug(format(c.channel(), "Handler is being applied: {}"), handler);
            }
            Mono.fromDirect(handler.apply(c.inbound(), c.outbound()))
                .subscribe(c.disposeSubscriber());
        }
    }
```
在这里，我们终于看到了subscribe实际调用的地方。数据源是用户自定义函数返回的Publisher，实际订阅者是`c.disposeSubscriber()`，针对TCP服务器来说，这个就是ChannelOperations。

但还有如下疑问：

* 这里的数据源为什么是Mono，而不是Flux，真正的数据源是什么？
* subscribe会不会阻塞当前的I/O线程？
* 回压机制体现在什么地方？

带着这些疑问，我们再深入看下整个反应式流的装配、订阅和运行过程。

## 反应式流的装配过程
要想弄清楚整个反应式流是如何装配的，我们需要知道`handler.apply(c.inbound(), c.outbound())`背后到底做了些什么？

这里的`c.inbound()`和`c.outbound()`分别返回的是`NettyInbound`和`NettyOutbound`，Connection实际是ChannelOperations，而ChannelOperations实现中这两个接口都是返回ChannelOperations本身：

```java
    @Override
    public NettyInbound inbound() {
        return this;
    }

    @Override
    public NettyOutbound outbound() {
        return this;
    }

```


我们以EchoServer例子中所示的用户自定义handler来分析整个装配过程：

```java
out.send(in.receive().retain());
```

`in.receive()`在ChannelOperations中实现如下：


```java
    
    // 这里的inbound实际是FluxReceive
    @Override
    public Flux<?> receiveObject() {
        return inbound;
    }

    @Override
    public ByteBufFlux receive() {
        return ByteBufFlux.fromInbound(receiveObject(), connection.channel()
                                                                  .alloc());
    }

```

可以看到这里的原始数据源是FluxReceive，紧接着用ByteBufFlux操作符对FluxReceive进行了修饰，作用是将数据转换成ByteBuf类型，`retain()`函数是增加ByteBuf的引用计数，防止被回收。

我们再来看下`out.send`做了什么工作，看下ChannelOperations的实现：

```java
    @Override
    public NettyOutbound send(Publisher<? extends ByteBuf> dataStream, Predicate<ByteBuf> predicate) {
        // 连接关闭时，直接发射错误事件
        if (!channel().isActive()) {
            return then(Mono.error(AbortedException.beforeSend()));
        }

        // 如果数据源是Mono类型，直接通过channel写出
        if (dataStream instanceof Mono) {
            return then(((Mono<?>)dataStream).flatMap(m -> FutureMono.from(channel().writeAndFlush(m)))
                                             .doOnDiscard(ByteBuf.class, ByteBuf::release));
        }

        // 否则通过MonoSendMany进行包装
        return then(MonoSendMany.byteBufSource(dataStream, channel(), predicate));
    }

```
从上述实现中，可以了解到最后会通过MonoSendMany对数据源进行包装，而MonoSendMany是负责将用户的数据发送到对端的，实际上不会产生任何数据到下游订阅者，所以将Flux的数据源包装成Mono类型，这样可以被ChannelOperations订阅从而触发整个反应式流。

所以，我们可以回答上面提到的第一个问题：

* 数据源为什么是Mono，而不是Flux，真正的数据源是什么？

真正的数据源是FluxReceive，因为最后通过MonoSendMany包装了原始的数据源，进行数据的发送处理，并不会有任何数据发送到下游订阅者，也就是ChannelOperations。

至此，我们得出整个装配完的反应式流如下图所示：

{% capture images %}
    /images/reactor_netty_assembly.png
{% endcapture %}
{% include gallery images=images%}

## 反应式流的订阅过程
为了回答接下来的两个问题，我们需要看下subscribe被调用之后发生了什么？

和一般的反应式编程一样，会进行subscribe -> onSubscribe -> request -> onNext的流程。

首先是subscribe过程，整个调用链路会经由MonoSendMany -> ByteBufFlux -> FluxReceive。(这里省略了Reactor内部的中间操作)

MonoSendMany的subscribe函数实现如下：

```java
    @Override
    public void subscribe(CoreSubscriber<? super Void> destination) {
        source.subscribe(new SendManyInner<>(this, destination));
    }
```

这里会将原始的CoreSubscriber也就是ChannelOperations封装成SendManyInner，以SendManyInner方式订阅原始数据源。SendManyInner就是MonoSendMany内部实现数据发送的订阅者。

BytebufFlux实现的subcribe函数没有做特殊处理，直接传递至原始数据源。也就是说真正的订阅和发布是在FluxReceive和SendManyInner之间实现的。

接下来，我们看下FluxReceive的subscribe实现：

```java
    @Override
    public void subscribe(CoreSubscriber<? super Object> s) {

        // 确保只被订阅一次
        if (once == 0 && ONCE.compareAndSet(this, 0, 1)) {
            if (log.isDebugEnabled()) {
                log.debug(format(channel, "{}: subscribing inbound receiver"), this);
            }

            // 整个数据流结束
            if (inboundDone && getPending() == 0) {
                if (inboundError != null) {
                    Operators.error(s, inboundError);
                    return;
                }

                Operators.complete(s);
                return;
            }

            receiver = s;

            // 开启onSubscribe流程
            s.onSubscribe(this);
        }
        else {
            if (inboundDone && getPending() == 0) {
                if (inboundError != null) {
                    Operators.error(s, inboundError);
                    return;
                }

                Operators.complete(s);
            }
            else {
                Operators.error(s,
                        new IllegalStateException(
                                "Only one connection receive subscriber allowed."));
            }
        }
    }
```
然后，是onSubscribe过程，FluxReceive实现了Subscription接口，会将自身传递给订阅者。这里，我们直接看SendManyInner的onSubscribe实现：

```java

        @Override
        public void onSubscribe(Subscription s) {
            // 只能被调用一次
            if (Operators.setOnce(SUBSCRIPTION, this, s)) {
                // 队列融合，使用上游发布者的队列，避免额外创建队列，详见操作符融合相关
                if (s instanceof QueueSubscription) {
                    @SuppressWarnings("unchecked") QueueSubscription<I> f =
                            (QueueSubscription<I>) s;

                    int m = f.requestFusion(Fuseable.ANY | Fuseable.THREAD_BARRIER);

                    // 同步融合
                    if (m == Fuseable.SYNC) {
                        sourceMode = Fuseable.SYNC;
                        queue = f;
                        terminalSignal = Completion.INSTANCE;
                        actual.onSubscribe(this);
                        trySchedule();
                        return;
                    }
                    // 异步融合
                    if (m == Fuseable.ASYNC) {
                        sourceMode = Fuseable.ASYNC;
                        queue = f;
                        actual.onSubscribe(this);
                        s.request(MAX_SIZE);
                        return;
                    }
                }

                queue = Queues.<I>get(MAX_SIZE).get();
                actual.onSubscribe(this);
                // 向上游请求MAX_SIZE的数据
                s.request(MAX_SIZE);
            }
            else {
                queue = Queues.<I>empty().get();
            }
        }
```
接着，就是是request过程，我们可以看到，SendMangInner请求的数据量是MAX_SIZE，默认是128条数据。这也是反应式编程的设计理念，通过request函数向上游传递下游需求。FluxReceive接收到下游的request后会做如下操作：


```java
    @Override
    public void request(long n) {
        if (Operators.validate(n)) {
            if (eventLoop.inEventLoop()) {
                this.receiverDemand = Operators.addCap(receiverDemand, n);
                drainReceiver();
            }
            else {
                eventLoop.execute(() -> {
                    this.receiverDemand = Operators.addCap(receiverDemand, n);
                    drainReceiver();
                });
            }
        }
    }

```
这里的`drainReceiver()`是一个loop，会将channel中的数据根据需求量发布给下游，这里会保证该函数是在eventLoop线程中执行的，eventLoop线程即时Netty中的I/O读写线程。

至此，可以回答上面提出的第二个问题：

* subscribe会不会阻塞当前的I/O线程？

其实整个subscribe过程都是在I/O线程中进行的，如果用户自定义handler中不包含阻塞的调用，整个过程会非常快，基本不会阻塞I/O线程。所以在反应式编程中要避免阻塞的调用，如果必须有阻塞调用，可以使用subscribeOn操作符，让阻塞操作在其他线程中执行。


## 反应式流的运行过程
我们再深入看下单个连接的反应式流是如何运行的，以及解答最后一个问题，回压机制是怎样的？

channel数据读取的入口是在`ChannelOperationsHandler.channelRead`中：


```java
    @Override
    @SuppressWarnings("FutureReturnValueIgnored")
    final public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg == null || msg == Unpooled.EMPTY_BUFFER || msg instanceof EmptyByteBuf) {
            return;
        }
        try {
            ChannelOperations<?, ?> ops = ChannelOperations.get(ctx.channel());
            if (ops != null) {
                // 直接调用传递给ChannelOperations.onInboundNext函数
                ops.onInboundNext(ctx, msg);
            }
            else {
                ...
            }
        }
        catch (Throwable err) {
            ...
        }
    }
```
`ChannelOperations.onInboundNext`则直接调用`FluxReceive.onInboundNext`函数发布数据，函数实现如下：


```java
    final void onInboundNext(Object msg) {
        if (inboundDone || isCancelled()) {
            ......
            return;
        }

        if (receiverFastpath && receiver != null) {
            ......
        }
        else {
            Queue<Object> q = receiverQueue;
            if (q == null) {
                // please note, in that case we are using non-thread safe, simple
                // ArrayDeque since all modifications on this queue happens withing
                // Netty Event Loop
                q = new ArrayDeque<>();
                receiverQueue = q;
            }

            ......

            q.offer(msg);
            drainReceiver();
        }
    }
```

FluxReceive会将接收到的数据放在自身的队列之中，然后调用`drainReceiver`循环。下面就是关键的`drainReceiver`实现：


```java

    final void drainReceiver() {
        // general protect against stackoverflow onNext -> request -> onNext
        if (wip++ != 0) {
            return;
        }
        int missed = 1;
        for(;;) {
            final Queue<Object> q = receiverQueue;
            final CoreSubscriber<? super Object> a = receiver;
            boolean d = inboundDone;

            if (a == null) {
                ......
            }

            long r = receiverDemand;
            long e = 0L;

            while (e != r) {
                ......

                try {
                    a.onNext(v);
                }
                finally {
                    try {
                        ReferenceCountUtil.release(v);
                    }
                    catch(Throwable t) {
                        inboundError = t;
                        cleanQueue(q);
                        terminateReceiver(q, a);
                    }
                }

                e++;
            }

            ......

            if (r == Long.MAX_VALUE) {
                ......
            }

            // 回压机制，当下游还有需求时，或者队列积压小于一定阈值时，开启channel的autoRead，否则关闭autoRead
            if ((receiverDemand -= e) > 0L || (e > 0L && q.size() < QUEUE_LOW_LIMIT)) {
                if (needRead) {
                    needRead = false;
                    channel.config()
                           .setAutoRead(true);
                }
            }
            else if (!needRead) {
                needRead = true;
                channel.config()
                       .setAutoRead(false);
            }

            missed = (wip -= missed);
            if(missed == 0){
                break;
            }
        }
    }

```
整个`drainReceiver`循环就是根据下游请求的需求量进行数据的发布，通过`onNext`函数传递给下游。循环中最关键的部分是回压机制是如何在连接层面生效的。

当满足下面两个条件时，会关闭channel的autoRead:

1. 下游需求<=0
2. 队列积压>=一定阈值，这里默认是32

通过此种机制，可以避免读取过多的数据到内存之中。这也是第三个问题的答案。

那什么时候下游的消费速度会跟不上上游的生产速度呢？

可能有两种情况，一种是用户自定义操作中有耗时的操作，另一种是网络对端接收速度跟不上，我们再看下SendManyInner在onNext中接收到数据是如何处理的：


```java
        @Override
        public void onNext(I t) {
            ......
            trySchedule();
        }

        void trySchedule() {
            ......

            try {
                if (eventLoop.inEventLoop()) {
                    run();
                    return;
                }
                eventLoop.execute(this);
            }
            catch (Throwable t) {
                ......
            }
        }

        @Override
        @SuppressWarnings("FutureReturnValueIgnored")
        public void run() {
            Queue<I> queue = this.queue;
            try {
                int missed = 1;
                for (; ; ) {
                    int r = requested;

                    while (Integer.MAX_VALUE == r || r-- > 0) {
                        I sourceMessage = queue.poll();

                        ......

                        int readableBytes = parent.sizeOf.applyAsInt(encodedMessage);

                        ......

                        pending++;

                        // 写入到内部缓冲
                        //"FutureReturnValueIgnored" this is deliberate
                        ctx.write(encodedMessage, this);


                        // 满足以下情况之一，则主动进行flush
                        // 1. 指定需要flush的
                        // 2. channel底层缓冲区已满，处于不可写状态
                        // 3. 待写入的数据量>缓冲区剩余容量
                        if (parent.predicate.test(sourceMessage) 
                            || !ctx.channel().isWritable() 
                            || readableBytes > ctx.channel().bytesBeforeUnwritable()) {

                            needFlush = false;
                            ctx.flush();
                        }
                        else {
                            needFlush = true;
                        }
                    }

                    // 异步flush情况
                    if (needFlush && pending != 0) {
                        needFlush = false;
                        eventLoop.execute(asyncFlush);
                    }

                    ......

                    // 决定下一次需要请求上游的量
                    int nextRequest = this.nextRequest;
                    if (terminalSignal == null && nextRequest != 0) {
                        this.nextRequest = 0;
                        s.request(nextRequest);
                    }

                    ......
                }
            }
            catch (Throwable t) {
                ......
            }
        }



```
所以在SendManyInner中会判断底层的写入缓冲区是否已满来决定要不要进行flush操作，默认是异步flush的，只有满足以下三种情况会进行主动flush:

1. 指定需要flush的
2. channel底层缓冲区已满，处于不可写状态
3. 待写入的数据量>缓冲区剩余容量

至此，整个反应式流及其回压机制已经梳理完毕。

## 番外篇-Netty之autoRead和isWritable
为什么Netty的autoRead参数可以控制channel的数据传输速率呢？

这背后其实是利用了TCP基于滑动窗口的流量控制机制，系统socket内核有receive buffer和send buffer两个缓冲区。对于接收方，如果应用层来不及消费导致receive buffer缓冲区满的情况下，TCP会告知对端减小发送窗口大小甚至停止发送。同样的，对于发送方，如果send buffer已满，异步情况下系统调用会告知应用层发送失败。

Netty通过autoRead参数来控制channel读事件的注册和移除。当autoRead打开时，会自动读取channel的数据给应用层，而当autoRead关闭时，则会移除读事件，此时TCP接收的数据会积压在receive buffer中，TCP的流控机制就会起效。

Netty的isWritable则是判断底层send buffer是否已经填满，如果send buffer已经填满，则应用层应该不再写入新数据。

## 总结
本文深入探究了Reactor Netty是如何在连接层面实现具有回压特性的反应式流，主要依赖三方面机制：

1. 反应式流的request接口
2. Netty的autoRead和isWritable参数
3. TCP基于滑动窗口的流量控制机制

这种回压特性让系统更加具有弹性和容错性，可以应对外面变化的流量，保持高度的自治，让系统显得更加的“聪明”，而不是在大流量面前直接陷入奔溃。



## 参考文献
1. [Reactor Netty](https://projectreactor.io/docs/netty/release/reference/index.html#getting-started-introducing-reactor-netty)
2. [Reactor Core](https://github.com/reactor/reactor-core)
3. [Netty](https://github.com/netty/netty)
4. [The Reactive Manifesto](https://www.reactivemanifesto.org/)
5. [EchoServer](https://github.com/reactor/reactor-netty/blob/master/reactor-netty-examples/src/main/java/reactor/netty/examples/tcp/echo/EchoServer.java)
6. [Spring WebFlux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux)




