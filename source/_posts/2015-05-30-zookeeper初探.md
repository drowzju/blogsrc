---
title: ZooKeeper初探
author: 梅溪
comments: true
layout: post
slug: 
tags : [分布式]
date: 2015-05-30 23:45:26
---


# 简介

我开始学习Hadoop之后，阅读各种资料时经常遇到的一个特性就是ZooKeeper。而且经常看到对ZooKeeper的功能有不同的描述，有的说它提供统一命名服务，有的说它提供状态同步服务，有的说它用于分布式系统的管理，有的说它用于分布式应用的配置项。多样的描述让我觉得很郁闷: ZooKeeper到底是什么？

其实这些说法都对。对ZooKeeper更准确的描述，应该说它是一个分布式协调服务。本文尝试介绍ZooKeeper的特征，原理，以及一些典型的应用场景。
<!--more-->
# 分布式集群和CAP

面对海量的计算或存储需求，采用分布式的计算机集群已经成为非常普遍的做法。可是分布式的数据存储带来的是系统的复杂度，以及操作一致性和服务可用性上的问题。

-   比如有多个数据副本的前提下，我们希望所有用户看到的内容相同，就要保证对数据的修改在所有副本生效并对
    用户可见；
-   比如一个系统中有主控单元存放元数据或者集中处理用户请求，那么就需要考虑主控单元故障后该如何保证副主控

单元接管服务，以及如何避免脑裂现象；

你会发现在分布式服务上修改一个再简单不过的配置都有可能发生不一致问题。

2000年PODC大会时Brewer提出了CAP理论，明确了一个大规模可伸缩网络服务不可能同时满足这三点：

-   Consistency: 一致性，就是一项操作的完成后应该所有用户都看以看到最新状态。可以理解为操作的原子性
-   Availability: 可用性，就是数据的读和写
-   Partition Tolerance: 分区容忍度。出现网络分区(即一部分和另外部分不可达)时不影响服务提供。

CAP理论已经得到了证实。当下常见的分布式计算和存储系统往往牺牲其中的某一项。

而ZooKeeper作为协调系统，是在部分牺牲可用性和分区容忍度的前提下，提供的是一致性的功能(虽然也不是严格一致性)。

这也是为什么这么多其他业务系统离不开ZooKeeper系统的原因：因为它提供了它们在某些方面所缺失的一致性。

下面我们就看看ZooKeeper的架构，以及它具体提供的功能。

# 体系架构

## 服务模型和数据模型

如图，一个完整的服务有一个Leader Server，多个普通Server(Learner)。用户请求可能定向到任何Server。读操作可以直接从任意Server获得响应；写操作则需要被请求Server重定向到Leader Server。

<img style="mergin:5px;" src="/images/ZK.png">

ZooKeeper会维护一个具有多层级的数据结构，每个节点叫做Znode。Znode 本身可以存储不超过1M的数据，也包含一种简单的ACL机制（和Unix文件的访问控制不太一样，子节点不会继承父节点）用于多类集群共用ZooKeeper时的访问控制。

ZooKeeper的核心就是这样一个精简的文件系统，看起来就像一个典型的多层目录结构。

<img style="mergin:5px;" src="/images/zkznode.jpg">

需要注意ZooKeeper是协调系统，应该尽量避免把它当存储系统用。

ZooKeeper对单个数据的写要求原子性，要么成功要么失败，不会部分写。只有Leader可以写，所有Leaner都可以读。

Znode的特征:
-   生命周期: 永久节点和临时节点
-   访问控制: ACL
-   Watches: 类似于监视器。增删改则触发通知。
-   顺序: SEQUENTIAL，节点自增属性，基于此属性ZooKeeper可以为节点名自动增加序号。

## ZAB广播

ZooKeeper 号称可以同时响应上万个客户端请求，强调的是它的高吞吐性能。(任意服务器都可响应读，这是高吞吐的重要原因。)

名为Zab的原子广播协议是实现这一点的核心。Zab有两种工作模式:

-   恢复模式: 选举。当服务启动或者Leader崩溃后进入。
-   广播模式: 同步。选举完成Leader能正常工作时的模式。

Leader选举过程有basic paxos和fast paxos两种算法。区别在于前者是有单独的线程发起让所有Server投票，后者是Server自荐。具体选举实现可以参见: <http://csrd.aliapp.com/?p=162>

Zab需要保证一致性和顺序性。一致性协议采用简单的多数投票仲裁。而顺序性由zxid保证。

zxid编号从字面看即所谓ZooKeeper Transaction ID。它在通信中用于保证事务的一致性。实现中是一个64位数字，高32位用于标志Leader，低32位用于递增计数。比如Client所连接的Server发生了切换，新Server就要比对Client
请求中的zxid，保证自己的数据不能比Client的老（若发现老可以从Leader重新同步）。简单的递增保证了事务的顺序性，这在网络协议中很常见。

## Leader 和Follower

Server有Leader和Learner之分。Learner又有Follower和Observer之分。

Leader很容易理解是系统的主。Follower和Observer的差别在于后者不参与投票，只是为了扩展系统的吞吐能力。

Leader 的主要职责是:

-   恢复数据
-   维持心跳，接收请求
-   提交写操作，延长Session时间等

Leader 处理的消息:

-   PING       心跳
-   REQUEST    请求。比如写请求，同步请求。
-   ACK        Follower 对提议的回复。超过半数
-   REVALIDATE 延长Session有效时间

Follower职责:

-   向Leader发送消息
-   接收、处理Leader消息
-   接收处理Client请求。如果是写操作需要发送给Leader
-   向Client返回操作结果

Follower处理的消息:

-   PING 心跳
-   PROPOSAL Leader发起的提案，所有Follower投票
-   COMMIT 服务器端最新通过的提案
-   UPTODATE 同步完成
-   REVALIDATE Leader反馈的REVALIDATE结果
-   SYNC  我们是分布式系统，一个问题就是不太可能满足数据的更新对Client的即时可见性。当对即时可见有要求
    时，client会发起SYNC请求，Follower需要响应此请求，从Leader获取最新数据并返回给Client。

从中我们可以看到，ZooKeeper一致性的本质是“多数派民主"。

## 容错性

ZooKeeper的容错恢复采取的是fuzzy snapshot加replay log机制。

fuzzy snapshot表示是一个很模糊的快照。为什么是模糊的？因为快照的记录这一动作显然不是原子的，在复制快照的过程中可能数据又不断会有变化。

replay log是操作的忠实记录。当需要恢复数据时，我用一个模糊的记录加一个忠实的重放，就能得到最终的正确数据。当然，做到这一点的前提是ZooKeeper对写操作原子性的要求，对数据重复操作多次也能得到同样结果，这体现了所谓 **幂等性** , 达到了所谓的 **最终一致性** 。

# 典型应用

## Leader Election

这可能是ZooKeeper用的最多的特性。因为分布式系统大多采用Master-Slave的主从结构。为了解决单点故障问题，使用ZooKeeper进行主的发现和选举功能。

做法可以是设置一临时节点用于存储Leader信息和其他扩展信息，然后所有非领导者观察此节点以明确领导者丢失或者变化的情况。

使用ZooKeeper来做主备的有HDFS的NameNode，HBase，Mesos，Yarn的RM……真的太多了。

## Configuration Management

这也是ZooKeeper的一大热门场景。利用znode节点的观察属性，使得配置的变化可以及时通知到客户端。

HBase，Kafka等许多系统多多少少都利用了ZooKeeper来保存配置或者运行信息。

## Group Membership

分布式系统的弹性计算能力很重要，成员的动态加入退出是集群管理者必须要识别的。ZooKeeper为可变化的组设置带Watch属性节点以供通知到监控进程，并可利用节点的SEQUENTIAL属性来自动命名Watch属性节点之下的新建子节点。当有成员加入退出时(即新子节点的创建或者子节点的删除)，监控进程得到通知，完成后续管理工作。

最典型应用就是Kafka，利用这一功能完成Broker和Consumer的动态管理。

## Task Assignment

这也是非常实用的场景。利用znode来记录任务的分配。监控进程Watch任务的执行情况，工作节点监控任务到来情况。通过记录和观察，能够比较简单地实现分布式系统的负载均衡。

最经典的应用还是Kafka，在消息服务器之间进行负载均衡。

## Locks And Double Barrier

利用创建的znode的SEQUENTIAL和Watch属性，可以构建出分布式系统的排他锁以及读写锁。

顺序锁(伪代码)：


    create my lock(znode) with SEQUENTIAL;
    while True:
        if my lock is minimum node:
        I got lock;
        break;
        else:
        watch the smaller lock and wait;


读写锁(伪代码):


    #写端:
    create my write lock(znode) with SEQUENTIAL;
    while True:
        if my lock is minimum node:
        I got lock;
        break;
        else:
        watch the nearest smaller lock and wait;

    #读端:
    create my read lock(znode) with SEQUENTIAL;
    while True:
        if no write lock lower than me:
        I got lock;
        break;
        else:
        watch he nearest smaller write lock and wait;



解锁都是删除节点。

所谓barrier，是指并发的进程或者线程需要达到某个地点后再推进，就类似一种屏障的效果。ZooKeeper实现双向的Barrier(双向意味着到达和离开都需要同步)也很简单，通过创建znode并判断子节点数目是否满足N或者0即可。

# 总结和回顾

我们回头再看CAP:

-   C: Zookeeper要求数据写操作原子。而多节点之间由投票机制保证一致性。
-   A: 应该说ZooKeeper的可用性满足情况还是比较好的。ZooKeeper标榜的就是高吞吐高性能，Yahoo!的数据是每
    秒过万个操作。可是需要注意这个其实也有局限，比如之前说到的，需要Sync同步才能保证数据最新。而且极端
    情况下投票机制仅保证2N+1中有N+1个正确Server。
-   P: 选举机制对P有一定程度的满足。可是节点多了各种操作耗时，选举本身也变成非常笨重的行为。

本文讲述了ZooKeeper的最基本原理和应用，水平有限，描述谬误之处欢迎指教，多谢。
