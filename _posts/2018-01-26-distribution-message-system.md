---
layout: post
title: 分布式消息系统的若干问题
description: "分布式消息系统"
modified: 2018-01-26
tags: [RocketMQ, Kafka]
categories: [Distributed Systems]
---

分布式消息系统，简单来说是实现消息的发布和订阅功能，可以实现系统间解耦、异步通信、分布式事务等功能，是现代大规模系统的重要组成部分。

如何实现一个分布式消息系统？

从功能上说，主要是实现消息的异步通信功能，也就是生产者消费者的发布订阅模式。

从分布式角度上说，要保证系统高可靠（数据一致性）、高可用（性能）、分区容忍性（网络异常、节点宕机）。

* 高可靠就是要保证数据的一致性，保证消息有序、正确、不丢。
* 高可用就是整个系统提供服务的时间比例，一般是3至5个9。同时也要保证系统吞吐率大，每秒能处理的消息数多，从生产到消费花费的时间少，消息及时性要好。
* 分区容忍性就是要允许网络存在异常或者部分节点宕机，系统仍然可以对外工作。


正如CAP理论，分区容忍性一般是必须满足的，而高可用和高可靠往往是鱼和熊掌不可兼得，实际应用时可以根据业务特性通过配置的形式进行折中。所以我们的目标就是实现基本功能，并尽可能以高性能的方式满足不同业务场景下的高可靠和高可用特性。

分布式消息系统大概可以分为以下三个核心问题：

* 消息生产消费的逻辑模型是什么？
* 消息的存储模型是什么？
* 消息高可用的方案是什么？

目前分布式消息系统的开源方案有很多，比如RocketMQ、Kafka、ActiveMQ，三者的对比可以参看[https://rocketmq.apache.org/docs/motivation/](https://rocketmq.apache.org/docs/motivation/)。RocketMQ是借鉴kafka设计的java版本的实现，两者大同小异，下面以RocketMQ为例分别讨论上述几个问题。

## 系统架构
系统架构在设计时需要考虑到可扩展，高可用。需要将功能隔离，实现每个功能组件化且可横向扩展。

### 系统角色
分布式消息系统中的角色可以分为如下：

* 生产者（producer），或是生产者集群（producer group），负责消息的生产服务。
* 消费者（consumer），或者是消费者集群（consumer group），负责消息的消费服务。
* 消息中介（broker），负责消息的路由服务。
* 服务注册和发现（nameserver），负责所有角色服务的注册和发现服务。zookeeper是这一服务的开源实现，kafka采用了这一方案，而RocketMQ则自己实现了一个简单的nameserver作为所有服务的注册和发现。

### 负载均衡
负载均衡通常是分布式系统需要考虑的地方，均衡系统中每个节点的负载，不至于某个节点负载过高而宕机。

producer不需要考虑负载的问题，毕竟消息生产没有什么复杂的处理逻辑，生产速度可以非常快，但是指定哪一个broker进行投递需要进行负载均衡。并发消费的情况下，可以采用轮询的方式逐一发送到不同的broker，使得broker负载均衡。而顺序消费的情况下，可以在业务上通过哈希算法，将同一哈希值的消息发送到同一个broker，保证消息顺序消费。

conusmer消费集群则需要考虑到负载的问题，由于消息的消费是重任务，如果负载过高，就会产生消息堆积的现象。消费者集群的负载均衡一般可以通过平均哈希和一致性哈希算法来实现，具体的负载均衡策略可以配置。

### 高可用
高可用主要是保证单点故障之后，系统仍然可以正常提供服务。首先是解决单点问题，所有角色都需要是集群模式，部署两台及两台以上机器。分布式消息系统中的关键角色是broker，解决了broker的高可用性，也就保证了整个系统的可用性。

RocketMQ的解决方案是将master/slave模式，master提供读写服务，slave提供只读服务，master同步本机数据至slave，类似于mysql的主从复制。同时部署多套master-slave机器，当master宕机之后，消费者从slave消费剩余消息。[RocketMQ的架构](https://rocketmq.apache.org/assets/images/rmq-basic-arc.png)如下：

{% capture images %}
	/images/rmq-basic-arc.png
{% endcapture %}
{% include gallery images=images caption="RocketMQ架构" cols=1 %}

master/slave方案一个缺点是如何选择slave并将其切换成为master。因为slave在同步过程中可能出现宕机以及同步不及时的问题，所以需要手动切换master，这样降低了系统可用性，增大了运维成本。

而Kafka基于zookeeper和ISR机制实现了分布式一致性。最新版本的rocketmq则基于raft实现了Dledger库来完成broker的commitLog的一致性。

### 高可靠
高性能取决于消息生产的速度和消费的速度，而高可靠则要确保消息不丢，这就意味着消息需要进行存储。而存储过程中的IO操作必然会影响系统吞吐量，所以系统在设计时需要在两者之间取舍，并且让存储模块设计的尽可能的高效。针对不同的使用场景，业务对性能和可靠的需求也不一样，可以通过配置的方式在性能和可靠之间选择。

broker可以同时支持同步存储和异步存储，对于需求高吞吐量而对可靠性需求不那么高的业务可以采用异步存储，比如日志分析。而对于可靠性需求高，无法容忍消息丢失的业务可以采用同步存储，比如金融类业务。

对于高可用中的主从复制同样也提供了同步复制和异步复制模式，可以根据需求进行配置。当然，也不是说同步的性能就很低，性能的对比都是相对的，只要可以完全满足业务的需求，那么系统设计就是合理的。

## 生产消费逻辑模型
消息的生产和消费需要满足针对不同的场景生产和消费不同类型的消息。故而衍生出topic的概念，生产者和消费者通过topic连接，而每个topic又可以关联多个队列（Queue），Queue是最细粒度的队列，一个队列由一个消费者进行消费，一个消费者可以消费多个队列。RocketMQ中为了再区分同一topic的消息，还设置了tag的概念，供消费者进行筛选和过滤。

其次是消费者的消费模式，一般有pull模式和push模式，两种模式各有优缺点。

* pull模式是consumer从broker拉数据进行消费，这样使得broker是无状态的，具体消费哪些消息可以由consumer决定，但略微会影响消息的实时性。
* push模式则是broker给consumer推数据进行消费，这种模式实时性好，但是对broker压力大，需要进行负载均衡、状态保存等工作

鉴于分布式消息系统都是服务端部署的，pull模式的时间间隔可以设置很小，近乎实时。RocketMQ采用了pull模式。

[RocketMQ的消费模型](https://rocketmq.apache.org/assets/images/rmq-model.png)如下：

{% capture images %}
	/images/rmq-model.png
{% endcapture %}
{% include gallery images=images caption="RocketMQ消费模型" cols=1 %}

## 消息存储模型
消息的存储模型可以说是分布式消息系统的关键之处，其设计决定了系统的吞吐量和可靠性。如果需要保证消息可靠、不丢，那么直接用内存的方式是不可行的，当broker异常退出则内存中的消息会丢失。而可靠的存储方式可以是DB存储，也可以是本地文件系统，出于高性能的考虑，使用本地文件系统可以达到非常高的吞吐量，RocketMQ和Kafka都是采用了这一方案。

RocketMQ文件存储方案的设计借鉴了kafka，但是解决了kafka在队列增长时性能急剧下降的问题。原因是在kafka中，每一个队列都单独有一个文件夹，每一个队列是顺序写的，但是当队列数目增长时，从操作系统的角度看会变成了随机写，从而造成了写性能的下降。[RocketMQ的文件模型](http://alibaba.github.io/RocketMQ-docs/document/design/RocketMQ_design.pdf)如下图所示：

{% capture images %}
	/images/rocketmq-file.png
{% endcapture %}
{% include gallery images=images caption="RocketMQ文件模型" cols=1 %}

RocketMQ将所有消息都顺序写入CommitLog文件，按大小（默认是1G）拆分成小文件，CommitLog存储了消息的所有内容。

而ConsumeQueue文件保存了消息在CommitLog中的偏移（Offset），ConsumeQueue和CommitLog一样也是按大小拆分。每个topic可以有多个ConsumeQueue。

这样从操作系统层面保证是顺序写的。虽然随机读消息需要先读ConsumeQueue，再读CommitLog，但是由于ConsumeQueue很小，利用操作系统的PageCache机制基本上和读写内存性能差不多。但这个设计带来的一个缺点是需要保证CommitLog和ConsumeQueue的一致性，实现上复杂了些。

### 检查点机制
RocketMQ中只要消息写入了CommitLog，可以认为消息不会丢失，除非磁盘损坏（这种情况也可以通过slave机器恢复）。

但是对于写入CommitLog的消息，对于消费者还处于不可见阶段，必须将消息再写入到ConsumeQueue，消费者才能真正消费。

RocketMQ通过采用异步的方式不断从CommitLog中读取消息并写入ConsumeQueue，在进程正常关闭的情况下，通过jvm的shutdownHook机制会保证CommitLog和ConsumeQueue的消息一致。

那么异常情况下，如何维护CommitLog和ConsumeQueue的一致性呢？

解决方案就是检查点（checkPoint）机制，记录CommitLog和ConsumeQueue最后一次刷盘的时间戳，并记录是否是异常退出，如果异常退出的话，下次启动从上一次刷盘的位置重新写入ConsumeQueue，这样保证了CommitLog和ConsumeQueue的最终一致性。




