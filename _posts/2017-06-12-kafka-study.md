---
layout: post
title: kafka数据可靠性的深入研究
categories: 分布式
description: kafka
keywords: Kafka
---

## Kafka简介

**前世今生：**Kakfa起初是由LinkedIn公司开发的一个分布式的消息系统，后成为Apache的一部分，它使用Scala编写，以可水平扩展和高吞吐率而被广泛使用。目前越来越多的开源分布式处理系统如Cloudera、Apache Storm、Spark等都支持与Kafka集成。

Kafka凭借着自身的优势，越来越受到互联网企业的青睐，唯品会也采用Kafka作为其内部核心消息引擎之一。Kafka作为一个商业级消息中间件，消息可靠性的重要性可想而知。如何确保消息的精确传输？如何确保消息的准确存储？如何确保消息的正确消费？这些都是需要考虑的问题。本文首先从Kafka的架构着手，先了解下Kafka的基本原理，然后通过对kakfa的存储机制、复制原理、同步原理、可靠性和持久性保证等等一步步对其可靠性进行分析，最后通过benchmark来增强对Kafka高可靠性的认知。。

![](/images/kafka.png)

如上图所示，Kafka体系由N个Producer，N个borker，N个Consumer，以及一个Zookeeper集群组成，Zookeeper在这里主要管理集群leader选举，负载均衡等工作，Producer将消息push到broker中，Consumer则相应在broker的订阅并pull消息。

## Topic&Partition&segment
熟悉MQ的朋友们应该都知道，topic是一类消息的集合，在kafka中，topic又可以分为若干个Partition，partition在存储层面是append log文件。新发布到此partition的消息都会被追加到log文件的尾部，同时以offset去唯一标记这条消息。这种写消息的方式是顺序写磁盘，这样保证了kafka的高吞吐量。

在topic被创建时，可以通过修改配置文件（$KAFKA_HOME/config/server.properties）来设置partition的数量，当一条消息被推送到broker中时，会根基partition的规则来选择具体的partition。如果这种规则足够合理，那么消息的分布将会非常均匀。
![](/images/kafka1.png)

## Kafka文件存储机制
在Kafka的文件系统里，topic下包含多个partition，每个partition作为一个目录，命名规则为topic名称+有序序号，有序序号从0开始到partition的总数量-1.

而在Kafka的设计中，每个partition又可以细分为多个segment段数据文件中，这些数据文件只需要支持顺序读写，这样极大的提高了磁盘的利用率。每个segment文件由.index和.log两个文件组成，分别为segment的索引文件和数据文件。partition的segment文件从0开始，后面的每个segment文件名都是该文件最后一条消息的offset值（偏移量）。


```java
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```


## Kafka复制原理及同步方式
![](/images/kafka2.png)

在partition中有2个概念要介绍一下，一个是LEO(Long End Offset)，及这个partition最后一条消息的偏移量，还有一个是HW（High Water Mark），是consumer能看到此partition的位置。

为了提高消息的可靠性，每个topic的partition有N个副本（replicas），N被称为是topic的复制因子（replica fator）的个数。Kafka通过多副本机制实现故障自动转移，这样保证在Kafka集群中一个broker失效，仍然保证集群服务的可用性。其中在这N个replicas中，有一个replica为leader，其他的都是follower，leader处理partition的所有的读写请求，并定期去同步follower上的数据。
如下图所示：
![](/images/kafka3.png)

Kafka提供了数据复制算法保证。当leader发生故障挂掉后，会有一个新的leader被选举并接受客户端消息的写入。Kafka确保在副本同步队列中选出一个leader。leader负责维护并跟踪ISR(In-Sync Replicas的缩写，表示副本同步队列)中follow的滞后状态。当一条消息被写入后，消息提交滞后被复制到其他的follower，所以消息复制的效率受到最慢的follower的影响。如果一个follower滞后太多或者已经失效挂掉，这个follower将从ISR中被踢出。

## ISR
上面提到的ISR极大的增强了Kafka的可用性。默认情况下Kafka的replica的值为1，即只有唯一的leader，但是一般我们都会设置成大于1的数，即3，这个3就是所有副本个数AR(Assigned Replicas)。ISR是AR的一个子集，由leader负责维护，follower从leader同步数据有一定的延迟（包括延迟时间replica.lag.time.max.ms和延迟条数replica.lag.max.messages两个维度, 当前最新的版本0.10.x中只支持replica.lag.time.max.ms这个维度），超过这个阈值，follower就会从ISR中被踢出，存在OSR中（Outof-Sync Replicas），同时新加入的follower也是在OSR中，AR=OSR+ISR.

Kafka 0.10.x版本后移除了replica.lag.max.messages参数，只保留了replica.lag.time.max.ms作为ISR中副本管理的参数。这样做是有原因的，因为replica.lag.max.messages这个参数有一定的歧义，这里指滞后的最大消息个数。这样设置在瞬时高峰流量时，producer每次发出的消息都超过4条，follower还没来得及同步就被判断滞后太多被踢出ISR，但是其实他们都是存活状态的，后面追赶上leader重新加入ISR。这样无疑增加了Kafka的性能损耗。


下图介绍了当producer产生消息后，ISR及HW和LEO的流转过程：

![](/images/kafka4.png)

由此可见Kafka的复制机制并不是完全同步，也并非异步方式。同步方式认为当所有的follower都复制完，这条消息才算commit，这样极大的影响了Kafka的吞吐量。而异步方式认为只要消息被写入leader，就认为被commit，这样会很容易造成数据的丢失，例如当一条消息被写入leader，follower还没来得及同步，这时候leader突然宕机，这条数据就消失了。

##  数据可靠性和持久性保证
当producer向leader发送数据时，可以通过request.required.acks参数来设置数据可靠性的级别：

1（默认）：这意味着producer在ISR中的leader已成功收到的数据并得到确认后发送下一条message。如果leader宕机了，则会丢失数据。
0：这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
-1：producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当ISR中只有leader时（前面ISR那一节讲到，ISR中的成员由于某些情况会增加也会减少，最少就只剩一个leader），这样就变成了acks=1的情况。

接下来对acks=1和-1的两种情况进行详细分析：

1. request.required.acks=1

producer发送数据到leader，leader写本地日志成功，返回客户端成功；此时ISR中的副本还没有来得及拉取该消息，leader就宕机了，那么此次发送的消息就会丢失。
![](/images/kafka5.png)
2. request.required.acks=-1

同步（Kafka默认为同步，即producer.type=sync）的发送模式，replication.factor>=2且min.insync.replicas>=2的情况下，不会丢失数据。

有两种典型情况。acks=-1的情况下（如无特殊说明，以下acks都表示为参数request.required.acks），数据发送到leader, ISR的follower全部完成数据同步后，leader此时挂掉，那么会选举出新的leader，数据不会丢失。

![](/images/kafka6.png)

## Leader选举
当一个topic的leader挂掉后，Kafka会从当前leader维护的ISR中选出一台机器作为新的leader。这样保证kafka需要的冗余度较低，当当前topic有f+1个副本时，最多可以容忍有f个服务器不可用。而当当前ISR中所有服务器中都挂掉后，有2种可行的方案，一种是等待ISR中的某一个服务恢复，另外是立即选出一个服务作为leader(ISR外），2种方案无非是响应时间和副本数据不一致中的抉择，各有利弊。

![](/images/kafka7.png)
如上图的场景，当前partition的副本数为3，replica-0，replica-1，replica-2分别处在3个broker中，ISR=(0，1)。此时设置设置request.required.acks=-1, min.insync.replicas=2，unclean.leader.election.enable=false。
如果此时replica-0挂掉，replica-1成为新的leader，但是由于min.insync.replicas=2，不能写入消息，可以取消息。此时可以尝试恢复replica-0，如果恢复成功，则系统正常，或者将min.insync.replicas=1，恢复写功能。

但是如果replica-0挂掉后，replica-1也挂掉，此时ISR=(1),leader=-1。此时修复replica-0和replica-1，如果都能起来，则恢复服务。但是replica-0恢复，但是replica-1没有恢复，此时不能选出新的leader，因为设置了unclean.leader.election.enable=false，leader只能从ISR中去挑选，此时可以将其改为true，同时min.insync.replicas=1，恢复write服务。
而replica-1恢复，replica-0不能恢复，则参考上面的情形。

## Kafka的发送模式