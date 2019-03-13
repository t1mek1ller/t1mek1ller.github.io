---
layout: post
title: 用Reactor进行反应式编程
description: "reactive programming with reactor"
modified: 2020-04-07
tags: [Reactor]
categories: [Reactor, Reactive Programming]
---

## 反应式编程是什么？
[维基百科](https://en.wikipedia.org/wiki/Reactive_programming)中对其的一句话定义如下：

>  Reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change.

首先反应式编程是一种声明式的编程范式。声明式意味着更关注结果的描述性，而不是具体的执行过程。比如SQL就是最常见的声明式语言，其他声明式编程语言还有逻辑编程语言Prolog和函数式编程语言Lisp等。声明式最直接的好处直观，对用户友好，用户只需要描述what，而不需要给出how。

其次，反应式编程关心的是数据流。[这篇简单易懂的介绍](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)将其描述如下：


> Reactive programming is programming with asynchronous data streams.

所有的输入都可以看做数据流，反应式编程是对这些数据流进行异步处理。如果说面向对象编程中一切都是对象，那么反应式编程中一切都是数据流。

最后，反应式编程中**反应**的意思是变化传播，整个系统可以快速检测到问题并作出响应，可以动态调整资源以适应变化的系统负载。[反应式宣言](https://www.reactivemanifesto.org/)中给出了反应式系统的整个架构如下：

{% capture images %}
    /images/reactive-traits.svg
{% endcapture %}
{% include gallery images=images cols=1 %}


## 为什么要进行反应式编程？
上图中也提到了反应式系统的价值，可以让系统快速响应（responsive），可维护（mantainable）和可扩展（extensiable）。而反应式编程可以让我们轻松构建这种系统。
具体的说，反应式编程可以有如下优势：

* 可读性，声明式的描述方式更加直观，避免陷入**Callback Hell**
* 高层次、高价值的并发抽象，使得异步编程更加容易
* 可复用、松耦合的组件
* 回压（backpressure），使得系统可以具有弹性（Resilient）和伸缩性（Elastic）


## 如何进行反应式编程？
有很多反应式编程库帮助开发者可以快速进行反应式编程，比如RxJava、Reactor。

David Karnok作为反应式编程大师，参与了RxJava和Reactor库的设计，给出了Reactive库的[代际发展](http://akarnokd.blogspot.com/2016/03/operator-fusion-part-1.html)：

* 第0代，JDK中的`java.util.Observable`API和基于callback（`addXXXListener`）API。这些API是典型的观察者模式，很难使用且不可组合。

* 第1代，以微软提出的Rx.NET库为代表，设计的`IObservable/IObserver`API，但是其整个设计是同步的，而且没有回压处理。

* 第2代，以RxJava 1.x和Akka为代表，实现了Reactive Extensions规范，也称之为[ReactiveX](http://reactivex.io/)，可以实现非阻塞的应用，并且实现了回压处理。

* 第3代，以RxJava 2.x和Reactor为代表，实现了[Reactive Streams API](https://www.reactive-streams.org/)，Reactive Streams提出了一致的反应式编程接口，使得各个实现了反应式库可以互相兼容，允许用户可以替换底层实现。Reactive Streams API已经成为JDK9的规范。

* 第4代，以RxJava 2.x和Reactor 3.0为代表，仍然沿用Reactive Streams API接口，但内部构建了通用的算子（operator）集合以及性能优化。

* 第5代，未来版本，计划支持双向reactive IO。


## Reactor是什么？
Reactor是Spring团队基于Reactive Streams API实现的第4代反应式编程库。相比RxJava，Reactor没有厚重的历史包袱，其全新的实现使得其概念更加清晰和易于使用。

Reactor支持JVM平台上完全的非阻塞反应式编程以及高效的回压处理，分别提出了`Flux`和`Mono`两种反应式的可组合的API来对数据流建模。

另外，Reactor团队还提供了Reactor-Netty的反应式网络库，可以快速实现具有回压机制的网络应用程序。

Spring 5.0的WebFlux也采用Reactor库实现反应式系统。

## 如何用Reactor进行反应式编程？

Reactor实现了Reactive Streams API，提供了反应式编程的最小接口集合，我们可以先review下这些API，对理解反应式编程理念有很大帮助。

* **Publisher**，发布者，可以发布无限序列的消息，可以根据订阅者的需求push消息，在任意时间点都可以动态服务多个订阅者，其接口如下：

{% highlight Java %}
public interface Publisher<T> {

    /**
     * 请求发布者开始发布消息
     * 可以被多次调用，每次调用相互独立，会新建一个Subscription(发布订阅上下文)
     * 每个Subscription只服务一个Subscriber
     * 一个Subscriber只能subscribe一个Publisher
     * 如果Publisher拒绝此次订阅或者订阅失败会触发onError回调
     */
    public void subscribe(Subscriber<? super T> s);
}
{% endhighlight %}



* **Subscriber**，订阅者，消费发布者发布的消息，可以进行订阅、消费消息、接收完成、接收错误等动作，其接口如下：

{% highlight Java %}
public interface Subscriber<T> {

    /**
     * 当调用Publisher.subcribe(Subscriber)时会被触发
     * Subscriber负责进行Subscription.request(long)调用，只有这个调用发生时才会开始真正的数据流
     * Publisher只会响应Subscription.request(long)操作
     */
    public void onSubscribe(Subscription s);

    /**
     * Publisher在接收到Subscription.request(long)调用时，通知订阅者进行消费
     */
    public void onNext(T t);

    /**
     * 失败终止状态通知
     */
    public void onError(Throwable t);

    /**
     * 成功终止状态通知
     */
    public void onComplete();
}
{% endhighlight %}

* **Subscription**，表示一个订阅者订阅发布者的上下文，用于控制数据交换，可以请求或者取消数据交换，其接口如下：

{% highlight Java %}
public interface Subscription {

    /**
     * 只有当此方法被调用时，发布者才会发布消息
     * 发布者只可以发布小于等于请求的数据量以保证安全性
     */ 
    public void request(long n);

    /**
     * 通知发布者停止发布数据并清理相关资源
     * 发布者不会立即停止发布数据，会等到上一次请求的数据量发布完成后结束
     */ 
    public void cancel();
}
{% endhighlight %}

* **Processor**，表示发布者和订阅者之间数据处理的阶段，可以看成是发布者和订阅者之间的管道，其接口如下：
{% highlight Java %}
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
{% endhighlight %}

下面介绍下Reactor中的关键特性。

### Mono和Flux
Mono和Flux是Reactor中的两个基本组件。Flux表示0或N个元素的异步序列，Mono表示0或1个元素的异步序列。

虽然Flux也可以表示0或1个元素的异步序列，但是Mono明确表示了序列的基数，使得语义更加清晰，比如一次Http请求只有一个响应，这种情况下用`Mono<HttpResponse>`表示比`Flux<HttpResponse>`更加合理。

Flux数据流处理示意图如下所示：

{% capture images %}
    /images/flux.png
{% endcapture %}
{% include gallery images=images cols=1 %}

`operator`是一系列应用在Flux数据流中的每一个元素上的算子，会生产出新的数据流。Flux除了发布数据流外，还会发布错误事件和成功完成事件标志整个流的结束。

相对应的，Mono数据流处理示意图如下所示：

{% capture images %}
    /images/mono.png
{% endcapture %}
{% include gallery images=images cols=1 %}


### Backpressure回压
回压是反应式编程中的核心概念。回压是指发布者的生产速率大于订阅者的消费速率时，下游对上游的压力反馈。反应式宣言中，对回压一词的描述如下：

> 当某个组件正竭力维持响应能力时， 系统作为一个整体就需要以合理的方式作出反应。 对于正遭受压力的组件来说， 无论是灾难性地失败， 还是不受控地丢弃消息， 都是不可接受的。 既然它既不能（成功地）应对（压力）， 又不能（直接地）失败， 那么它就应该向其上游组件传达其正在遭受压力的事实， 并让它们（该组件的上游组件）降低负载。 这种回压（back-pressure）是一种重要的反馈机制， 使得系统得以优雅地响应负载， 而不是在负载下崩溃。 回压可以一路扩散到（系统的）用户， 在这时即时响应性可能会有所降低， 但是这种机制将确保系统在负载之下具有回弹性 ， 并将提供信息，从而允许系统本身通过利用其他资源来帮助分发负载，参见弹性。

其核心是订阅者需要一种机制主动告知发布者自己的需求，从`Subscription`的接口也可以看出，通过`Subscription.request`告知`Publisher`需求量以避免`Publisher`生产过快从而导致buffer溢出。回压机制的引入将push模式变成push-pull模式。


Reactor中的回压机制的一个例子如下：

{% highlight Java %}
public void testBackPressure2() throws  Exception{
    Flux<Long> fluxInterval = Flux.interval(Duration.ofMillis(100));

    CountDownLatch latch = new CountDownLatch(1);
    fluxInterval.log().publishOn(Schedulers.newParallel("pub"), 1).subscribe(new BaseSubscriber<Long>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            System.out.println("onSubscribe Thread: " + Thread.currentThread().getName());
            request(1);
        }

        @Override
        protected void hookOnNext(Long value) {
            System.out.println("Consume: " + value + " Thread: " + Thread.currentThread().getName());

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            request(1);
        }

        @Override
        protected void hookOnError(Throwable throwable) {
            throwable.printStackTrace();
            latch.countDown();
        }

        @Override
        protected void hookOnComplete() {
            latch.countDown();
        }
    });
    assertTrue(latch.await(100L, TimeUnit.SECONDS));
}
{% endhighlight %}

我们用` Flux.interval(Duration.ofMillis(100))`以每100ms的速率生产一个Long值，并在`BaseSubscriber.hookOnNext`中执行`Thread.sleep(1000)`以每1秒的速率消费一个Long值。需要注意的是为了防止消费和生产在同一线程中，调用了`publishOn`来指定消费线程池，并指定`prefetch`为1，表示1秒只请求一个值。
显而易见，此时消费速率跟不上生产速率。这种情况下，`interval`下会抛出异常，上面代码的输出日志如下：

{% highlight Java %}
22:30:24.225 [Test worker] INFO  reactor.Flux.Interval.1 - onSubscribe(FluxInterval.IntervalRunnable)
onSubscribe Thread: Test worker
22:30:24.240 [Test worker] INFO  reactor.Flux.Interval.1 - request(1)
22:30:24.347 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(0)
Consume: 0 Thread: pub-1
22:30:24.454 [parallel-2] ERROR reactor.Flux.Interval.1 - onError(reactor.core.Exceptions$OverflowException: Could not emit tick 1 due to lack of requests (interval doesn't support small downstream requests that replenish slower than the ticks))
22:30:24.461 [parallel-2] ERROR reactor.Flux.Interval.1 - 
reactor.core.Exceptions$OverflowException: Could not emit tick 1 due to lack of requests (interval doesn't support small downstream requests that replenish slower than the ticks)
    at reactor.core.Exceptions.failWithOverflow(Exceptions.java:234) ~[main/:na]
    at reactor.core.publisher.FluxInterval$IntervalRunnable.run(FluxInterval.java:130) ~[main/:na]
    at reactor.core.scheduler.PeriodicWorkerTask.call(PeriodicWorkerTask.java:59) ~[main/:na]
    at reactor.core.scheduler.PeriodicWorkerTask.run(PeriodicWorkerTask.java:73) ~[main/:na]
    at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:485) ~[na:na]
    at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:278) ~[na:na]
    at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:269) ~[na:na]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1129) ~[na:na]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:603) ~[na:na]
    at java.lang.Thread.run(Thread.java:748) ~[na:1.8.0_172]
{% endhighlight %}

从上面日志可以看到，只消费了`0`之后就抛出异常了。那这种情况下要如何处理呢？可能如下几种选择：

1. 提高消费能力
2. 降低生产速率
3. 只消费部分数据，有若干选择策略，比如丢弃、忽略、抛异常、取最新数据

实际情况下，1和2都很难改变，为了让系统可以正常运行，只能选择3来保证部分可用（正如反应式宣言中的Resilient）。比如我们将上面的例子改成如下：

{% highlight Java %}

fluxInterval.log()
    .onBackpressureDrop(
        drop -> System.out.println("Drop: " + drop + " Thread: " + Thread.currentThread().getName()))
    .publishOn(Schedulers.newParallel("pub"), 1).subscribe(...);

{% endhighlight %}

添加`onBackpressureDrop`策略，丢弃多余的数据，可以使整个程序仍然可用。此时部分输出日志如下：

{% highlight Java %}
22:46:21.537 [Test worker] INFO  reactor.Flux.Interval.1 - onSubscribe(FluxInterval.IntervalRunnable)
onSubscribe Thread: Test worker
22:46:21.556 [Test worker] INFO  reactor.Flux.Interval.1 - request(unbounded)
22:46:21.659 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(0)
Consume: 0 Thread: pub-1
22:46:21.762 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(1)
Drop: 1 Thread: parallel-2
22:46:21.861 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(2)
Drop: 2 Thread: parallel-2
22:46:21.960 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(3)
Drop: 3 Thread: parallel-2
22:46:22.061 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(4)
Drop: 4 Thread: parallel-2
22:46:22.159 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(5)
Drop: 5 Thread: parallel-2
22:46:22.258 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(6)
Drop: 6 Thread: parallel-2
22:46:22.362 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(7)
Drop: 7 Thread: parallel-2
22:46:22.461 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(8)
Drop: 8 Thread: parallel-2
22:46:22.562 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(9)
Drop: 9 Thread: parallel-2
22:46:22.660 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(10)
Drop: 10 Thread: parallel-2
22:46:22.760 [parallel-2] INFO  reactor.Flux.Interval.1 - onNext(11)
Consume: 11 Thread: pub-1

......
{% endhighlight %}

从上面输出日志可以看到，`Flux.inteval`仍然按照既定速率生产数字0~11，但是消费者只消费了数字0和数字11，其他数字都被丢弃了。



### Scheduler线程调度
默认情况下，作用在`Flux`和`Mono`上的算子执行的线程是延用调用链上前一个算子所执行的线程。所以，没有任何处理的情况下，所有操作都是在`subscribe`调用线程上执行的。

Reactor提供了两种方法来设置执行的上下文：`PublishOn`和`SubscribeOn`。

`PublishOn`是用来指定后续算子执行所在线程，具有以下性质：

* 只影响`PublishOn`调用之后的算子
* 切换执行上下文到指定的`Scheduler`
* 影响`Subscriber.onNext`方法执行所在线程。比如上面例子中，`sleep`是在PublishOn中指定的线程中执行的
* 除非调用链后面有额外指定`PublishOn`，否则沿用前面的`Scheduler`

`SubscribeOn`是用来指定整个调用链上下文的执行线程，具有以下性质：

* 影响整个调用链，调用链中最早出现的`SubscribeOn`作为最终的线程配置
* 不影响之后出现的`PublishOn`配置

下面举一个例子说明以上原则：
{% highlight Java %}
@Test
public void testScheduler() throws Exception{
    CountDownLatch count = new CountDownLatch(1);
    Scheduler s1 = Schedulers.newParallel("scheduler-A", 4);
    Scheduler s2 = Schedulers.newParallel("scheduler-B", 4);
    Disposable disposable = Flux
            .range(1, 2)
            .log()
            .map(i -> {
                System.out.println("map-1: " + "value: " + i + " thread: [" + Thread.currentThread().getName() + "]");
                return i; })
            .subscribeOn(s1)
            .map(i -> {
                System.out.println("map-2: " + "value: " + i + " thread: [" + Thread.currentThread().getName() + "]");
                return i; })
            .publishOn(s2)
            .map(i -> {
                System.out.println("map-3: " + "value: " + i + " thread: [" + Thread.currentThread().getName() + "]");
                return i; })
            .subscribe(
                    i -> { 
                        System.out.println("subscribe: " + "value: " + i + " thread: [" + Thread.currentThread().getName() + "]");
                    },
                    e -> {
                        System.out.println("onError");
                        count.countDown();},
                    () -> {
                        System.out.println("onComplete");
                        count.countDown();
                    });

    count.await();
}
{% endhighlight %}

上面用了3个`map`输出当前执行线程，map-1和map-2之间调用`subcribeOn`指定执行线程池为scheduler-A，而在map-2和map-3之间调用`publishOn`指定线程池为scheduler-B。我们可以看到以下输出结果：

{% highlight Java %}
23:25:54.072 [scheduler-A-2] INFO reactor.Flux.Range.1 - | onSubscribe([Synchronous Fuseable] FluxRange.RangeSubscription)
23:25:54.075 [scheduler-A-2] INFO reactor.Flux.Range.1 - | request(256)
23:25:54.075 [scheduler-A-2] INFO reactor.Flux.Range.1 - | onNext(1)
map-1: value: 1 thread: [scheduler-A-2]
map-2: value: 1 thread: [scheduler-A-2]
23:25:54.076 [scheduler-A-2] INFO reactor.Flux.Range.1 - | onNext(2)
map-1: value: 2 thread: [scheduler-A-2]
map-3: value: 1 thread: [scheduler-B-1]
map-2: value: 2 thread: [scheduler-A-2]
subscribe: value: 1 thread: [scheduler-B-1]
map-3: value: 2 thread: [scheduler-B-1]
subscribe: value: 2 thread: [scheduler-B-1]
23:25:54.076 [scheduler-A-2] INFO reactor.Flux.Range.1 - | onComplete()
subscribe: onComplete thread: [scheduler-B-1]
{% endhighlight %}

从上述日志可以发现，Flux.range以及map-1和map-2都是在scheduler-A-2线程中执行的，说明`subscribeOn`的确会影响调用链上的所有算子，而和其位置无关。

但是map-3和subscribe是在scheduler-B-1中执行的，说明`publishOn`只会影响其后续的算子，而不会影响其前置的算子。

另外，我们注意到虽然通过`SubscribeOn`和`PublishOn`指定了算子的线程池，但`Flux`流中所有元素都是由算子选择的同一个线程进行处理的。如果需要流中的元素需要实现真正的并行执行，可以用`Flux.parallel`转成并行数据流，并用`runOn`指定线程池，如下面例子所示：
{% highlight Java %}
@Test
public void testParallelFlux() throws Exception{
    CountDownLatch count = new CountDownLatch(1);
    Scheduler s1 = Schedulers.newParallel("scheduler-A", 4);
    Disposable disposable = Flux
            .range(1, 5)
            .parallel(4)
            .runOn(s1)
            .map(i -> {
                System.out.println("[" + Thread.currentThread().getName() + "]" + " map-1 " + " value: " + i);
                return i;
            })
            .sequential()
            .subscribe(
                    i -> {
                        System.out.println("[" + Thread.currentThread().getName() + "]" + " subscribe" + " value: " + i);
                    },
                    e -> {
                        System.out.println("[" + Thread.currentThread().getName() + "]" + " subscribe.onError" );
                        count.countDown();
                        },
                    () -> {
                        System.out.println("[" + Thread.currentThread().getName() + "]" + " subscribe.onComplete");
                        count.countDown();
                    });
    count.await();
}
{% endhighlight %}

上面例子用`parallel(4)`指定并行数为4,表示生产4个数据流，输出如下所示：
{% highlight Java %}
[scheduler-A-3] map-1  value: 3
[scheduler-A-1] map-1  value: 1
[scheduler-A-4] map-1  value: 4
[scheduler-A-1] map-1  value: 5
[scheduler-A-2] map-1  value: 2
[scheduler-A-3] subscribe value: 3
[scheduler-A-3] subscribe value: 1
[scheduler-A-3] subscribe value: 2
[scheduler-A-3] subscribe value: 4
[scheduler-A-3] subscribe value: 5
[scheduler-A-3] subscribe.onComplete
{% endhighlight %}

我们可以看到生产出的5个数字在算子map中是在不同的线程执行的。另外，我们在`subscribe`前调用了`sequential`，这个方法的作用是让之前4个并行的数据流重新合并成一个，所以日志中`subscribe`都是在一个线程中执行的。如果不调用`sequential`，则会有4个`onComplete`状态通知。



## 参考文献
1. [The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
2. [The Reactive Manifesto](https://www.reactivemanifesto.org/)
3. [Hands-On Reactive Programming with Reactor](https://learning.oreilly.com/library/view/hands-on-reactive-programming/9781789135794/)
4. [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference)
5. [Reactive programming](https://en.wikipedia.org/wiki/Reactive_programming)
6. [Advanced Reactive Java](http://akarnokd.blogspot.com/2016/03/operator-fusion-part-1.html)
7. [ReactiveX](http://reactivex.io/)
8. [Reactive Streams](https://www.reactive-streams.org/)



