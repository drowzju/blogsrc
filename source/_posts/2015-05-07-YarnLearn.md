---
title: 读书笔记：《Hadoop技术内幕——Yarn》
author: 梅溪
comments: true
layout: post
slug: 
tags : [分布式, 大数据]
date: 2015-05-07 13:43:00+00:00
---


# 设计理念和架构

## 产生背景

### MRv1局限性

-   扩展性差               JobTracker同时兼备资源管理和作业控制，大瓶颈
-   可靠性差               master 单点故障导致集群不可用
-   资源利用率低           slot的力度太粗，map slot和reduce slot的划分僵硬
-   无法支持多种计算框架    内存，流式，迭代式

最主要还是jobtracker 负担太重，资源管理和应用程序控制没有分离。

以MR为核心还是以YARN 为核心？ 我觉得随着新的计算框架的出现和流行，目前的Hadoop大生态圈应该是以YARN为核心。
<!--more-->
### 多种计算框架并用

多种计算框架的资源统一使用，任务级别的资源隔离-&#x2014;>轻量级弹性计算平台, Yarn

共享集群的模式好处(对比一种框架一个集群):
-   资源利用率高
-   运维成本低
-   数据共享

对比MRv1 和YARN如图：

<img style="mergin:5px;" src="/images//yarn-mrv1.PNG">


## YARN基本架构

基本设计思想是把JobTracker拆分成全局的资源管理器RM和每个应用程序独有的AM

<img style="mergin:5px;" src="/images//yarn-structure.PNG">

### M-S结构

-   Master: ResourceManager
-   Slave : NodeManager

用户提交应用时，提供AM，它向RM申请资源，并要求NM启动可以占用一定资源的任务

### RM

-   Scheduler: 分配Resource Container(Mem, CPU, disk, Net)&#x2026;
-   Applications Manager(ASM): 管理系统中所有应用程序

### AM

每个用户提交的应用程序包含一个AM，功能:

-   与RM协商获得container
-   任务内部分配
-   与NM通信:start, stop
-   监控任务运行状态

### NM

每个节点上的资源和任务管理器:

-   定时向RM汇报本节点上的资源使用情况和各个container的运行状态
-   接收处理AM的container start/stop 请求

### Container

是对计算资源的抽象。每个任务分配一个container。

注意container和MRv1的slot概念不同，是动态资源划分单位，目前只支持CPU和内存两种资源，使用了轻量级的资源隔离机制Cgroups。

## YARN通信协议

通过RPC，采用拉取模型。

任何两个需要通信的模块间有且仅有一个RPC协议，通信双方一个Client，一个Server，client主动连接server。

## YARN工作流程

-   短应用程序:MR作业，Tez DAG作业
-   长应用程序:通常是服务。Storm, HBase

工作步骤:

-   阶段一  启动ApplicationMaster
    -   用户提交AM，启动AM的命令和用户程序
    -   RM为该应用程序分配第一个Container，并和对应NM通信，在此Container中启动AM
-   阶段二  任务执行
    -   AM向RM注册(后面用户可以通过RM查看应用程序状态)，为各个任务申请资源，并监控运行状态直到结束
    -   重复以下步骤:
        -   AM轮询，向RM申请、领取资源
        -   AM拿到资源则要求对应NM启动任务
        -   NM设置运行环境，启动任务
        -   各个任务向AM汇报状态和进度
    -   AM向RM注销并关闭自己

# 核心设计

## 基础库

-   Protocol Buffers: 一种轻便高效的结构化数据存储格式（对比XML）  YARN的RPC协议全由它定义  向后兼容如
    何保证？
-   Avro   YARN将之作为日志序列化库使用。功能和Protocol Buffers有重叠(RPC辅助)
-   RPC    透明性，高性能，可控性  基于MRv1，使用了Protocol Buffers做默认序列化方法
-   服务库和事件库  把对象服务化。对象间用事件通信，不是直接函数调用，同步变异步，降了耦
-   状态机库 采用状态机描述对象状态和迁移

# YARN 应用程序设计

编写YARN应用涉及3个RPC协议:

-   ApplicationClientProtocol
-   ApplicationMasterProtocol
-   ContainerManagementProtocol

## 客户端设计

客户端提交一个应用程序分两步:
-   getNewApplication 通过RPC从RM获得唯一的application ID
-   submitApplication 提交ApplicationMaster

编程库: org.apache.hadoop.yarn.client.YarnClient

## AM 设计

分两个部分

### AM-RM 编写流程

涉及三个步骤:

-   registerApplicationMaster
-   allocate 申请资源
-   finishApplication 执行完毕并退出

### AM-NM 编写流程

也是三个步骤:

-   startContainer 将申请到的资源二次分配给内部的任务
-   getContainerStatus 向NM询问Container运行状态
-   stopContainer 释放资源

运行过程中，用户可以getApplicationReport, 可以forceKillApplication

# RM 剖析

RPC:

-   ResourceTracker             Pull模型  NM总是周期性地向RM发起请求，汇报和领取任务
-   ApplicationMasterProtocol   注册、申请资源和释放资源
-   ApplicationClientProtocol   提交应用，查询状态和控制

RM功能:

-   和client交互，处理请求
-   启动和管理ApplicationMaster
-   管理NM，接收资源汇报并下发管理指令
-   资源管理和调度

RM组成:

-   用户交互模块
-   NM管理模块
-   AM管理模块
-   Application管理模块
-   状态机管理模块
-   安全管理模块
-   资源分配模块

<img style="mergin:5px;" src="/images//yarn-rm.PNG">

## 用户交互模块

ClientRMService, AdminService， 不同的通信通道

## AM管理

三个服务: 

-   ApplicationMasterLauncher 收到事件去NM启动或者停止ApplicationMaster
-   AMLivelinessMonitor       周期性遍历所有AM，看是否汇报心跳。未汇报的AM的所有正运行的Container被标
    记为失败。不是直接重新执行，而是告诉AM。若AM本身运行失败则RM重新申请资源。
-   ApplicationMasterService  处理来自AM的请求:注册、心跳和清理

AM 运行时周期地调用allocate,和RM通信，有三个作用:请求资源，获取新分配的资源，形成周期心跳

## NM管理

-   NMLivelinessMonitor     保活 没有心跳则认为NM上所有Container失败，也是不会直接重新执行，而是告诉AM
-   NodeListManager         类似于黑名单，白名单管理
-   ResourceTrackerService  处理NM的请求，注册和心跳

## Application管理

-   ApplicationACLsManager     管理应用程序的访问权限（查看和修改）
-   RMAppManager               负责应用程序的启动和关闭，还要处理收尾。
-   ContainerAllocationExpirer 不允许AM长时间不使用Container，这是回收机制。

## 状态机管理

### RMApp

维护一个Application生命周期的数据结构。实现是RMAppImpl类，维护一个Application状态机，和Application基本信息，以及所有运行尝试。

### RMAppAttempt

维护运行尝试的生命追的数据结构。

RMAppAttemptImpl本质上是维护的ApplicationMaster 生命周期

### RMContainer 状态机

RMContainerImpl 维护了Container状态机。

### RMNode 状态机

略（太烦）

## 安全管理

认证和授权：

-   Kerberos 一种基于第三方服务的认证协议
-   Token 一种基于共享密钥的双方身份认证机制
-   Principal Hadoop中被认证或者授权的主体

## 容错

-   AM  失败重启
-   NM  失败告诉AM
-   Container  空闲回收，失败通知AM
-   RM  well&#x2026;

### Hadoop HA

通常采用热备，一个Active若干Standby。基本思路还是Active Master把信息写入一个共享存储系统，Standby 读取信息。需要主备切换时，Standby需要先保证信息完全同步后，再切换到Master.

手动模式和自动模式

Zookeeper 核心功能是通过维护一把全局锁控制整个集群有且只有一个Active Master

YARN RM 的HA实现非常轻量（已分配的AM的资源信息和NM的资源使用信息通过心跳汇报机制重构）
保存client提交信息和实例创建（Attempt）信息。应用程序运行结束后，保存的信息将被移除。
RM重启后，加载之前保存的信息，创建RMApp对象和RMAppAttempt对象， 但是:

-   若发现NM不存在则要求重启
-   若发现AM不存在则RemoteException导致AM异常退出
-   NM重启并向RM注册后，一切照旧

# 资源调度器

是RM中的一个插拔式服务组件

## 背景

FIFO，适合批处理。Hadoop一开始考虑作业以批处理为主。

两种设计思路，一种是在一个物理集群上虚拟多个Hadoop集群，各自拥有全套独立的Hadoop服务，典型是HOD；

另一种是跨站调度器，支持多个队列多用户，按照需求对用户或者应用分组，分配不同用量，添加约束避免独占:

-   Capacity Scheduler(Yahoo!)
-   Fair Scheduler(Facebook)

YARN的资源调度器，以层级队列方式组织资源的。

作业类型:

-   批处理作业        耗时长，数据挖掘，机器学习
-   交互式作业        Hive（SQL），立刻返回结果
-   生产性作业        统计值计算，垃圾数据分析&#x2026; 我理解是CPU占用率高

另外，对硬件资源要求不同，有CPU密集型和I/O密集型作业。

## HOD

HOD的主要问题是:

-   多集群带来运维难度
-   利用率低下
-   为解决虚拟集群回收后数据丢失的问题，通常使用一个外部的公用HDFS，带来丧失部分数据本地特性的问题

## YARN资源调度器

本身是插拔式的，RM初始化时根据用户的配置创建一个资源调度器对象

实质上是个事件处理器:

-   NODE\_REMOVED
-   NODE\_ADDED
-   APPLICATION\_ADDED      RM要为每个Application维护一个独立的数据结构
-   APPLICATION\_REMOVED
-   CONTAINER\_EXPIRED
-   NODE\_UPDATE            最重要的事件，收到NM的汇报信息后出发，可能有新的Container得到释放，会触发资
    源分配

<img style="mergin:5px;" src="/images//yarn-scheduler.PNG">

资源表示

-   内存
-   单位物理内存对应最多可使用的虚拟内存量
-   可分配的虚拟CPU个数  允许把物理CPU划分成若干个虚拟CPU（考虑到CPU异构）

我的问题是虚拟CPU不到一个物理CPU，在实现上是如何使用？是按照CPU占用率统计来估算的？

调度语义:

-   请求特定节点的特定资源量
-   请求某个特定机架上的特定资源量
-   某些节点加入黑名单，不用它的资源
-   请求归还某些资源

不支持的调度语义:

-   请求任意节点或者任意机架的特定资源量
-   请求一组或几组特定资源？？
-   CPU性能要求，绑定
-   动态调整Container资源

双层调度模型:

-   RM的资源调度器把资源分配给AM
-   AM把资源分配给各个任务

资源不足时，增量资源分配(incremental placement)还是一次性分配(all-or-nothing)?
前者有浪费，后者易饿死。YARN采用前者。

YARN和Mesos都采用了DRF算法(Dominant Resource Fairness)

关于DRF，所有的中文书都用的这个例子，出自这里：<https://www.cs.berkeley.edu/~alig/papers/drf.pdf>

<img style="mergin:5px;" src="/images//drf.PNG">

DRF first picks B to run a task. As a result, the shares of B become (3/9,1/18), and the dominant
share becomes max(3/9, 1/18) Next, DRF picks A , as her dominant share is 0. The process continues
until it is no longer possible to run new tasks. In this case, this happens as soon as CPU has been
saturated.

资源抢占

-   如何决定是否抢占某个队列的资源？ 按照minresource权重计算应得资源比例，然后队列间均衡。
-   如何使资源抢占的代价最小？首先是选择优先级低的Container，然后不是直接杀死，而是告知AM，若AM未能杀
    掉才由RM杀。

## 层级队列管理机制

-   子队列
    -   队列可以嵌套
    -   用户只能把应用程序提交到最底层的队列，即叶子队列
-   最小容量
    -   每个子队列有“最少容量比”属性，表示可以使用父队列的容量的百分比
    -   调度器优先选择当前资源使用率最低的队列
    -   最少容量并不意味着永远保留
-   最大容量
    -   为了防止队列超量使用资源设置上限
    -   默认无限大

用户权限管理系统资源管理

队列命名: ROOT.C1.C11

## Capacity Scheduler

-   容量保证: 为每个队列设置资源最低保证和资源使用上限
-   灵活性: 队列资源有剩余可以暂时共享给需要资源的队列
-   多重租赁
-   安全保证
-   动态更新配置文件

资源调度中，YARN采用了三级资源分配策略，即队列、应用程序和container三级。

## Fair Scheduler

和Capacity Scheduler 类似，以队列为单位划分资源，也设定资源最低保证和上限，也有剩余资源共享……

不同处:

-   强调资源公平
-   资源抢占，会杀死任务(先等待再强制回收)
-   基于任务数目的负载均衡
-   调度策略配置灵活
-   小应用响应快（因为公平，小作业快速获得资源立刻运行完成）

两种调度方式的实现对比如下：
<img style="mergin:5px;" src="/images//yarn-fair-cap.PNG">

# NM 剖析

## 概述

RPC协议

-   ResourceTrackerProtocol             RM(S)-&#x2014;NM(C)
-   ContainerManagementProtocol         NM(S)-&#x2014;AM(C)

内部结构

-   NodeStatusUpdater  与RM通信
-   ContainerManager   核心组件，管理Container
    -   RPC Server
    -   ResourceLocalizationService
    -   ContainersLauncher   维护线程池以并行完成Container相关操作
    -   AuxServices          允许用户通过配置附属服务的方式扩展自己的功能
    -   ContainersMonitor 监控Container的资源使用量
    -   LogHandler
    -   ContainerEventDispatcher    把ContainerEvent类型的事件调度给对应Container的状态机实现类
    -   ApplicationEventDispatcher  将ApplicationEvent调度给对应Application的状态机ApplicationImpl
-   ContainerExecutor             和底层OS交互，安全存放Container需要的文件和目录，以安全方式启动清除
    Container对应的进程
-   NodeHealthCheckerService      检查节点监控情况
-   DeletionService               文件删除服务异步
-   Security
    -   ApplicationACLsManager
    -   ContainerTokenSecretManager
-   WEbServer

<img style="mergin:5px;" src="/images//yarn-nm.PNG">

## 节点健康状况检测

可以通过自定义shell脚本检查健康状况。YARN仅对内存和CPU做了隔离，磁盘IO和网络可能任务间有干扰。

## 分布式缓存机制

类似于MRv1中的DistributedCache，外部文件资源自动透明地下载并缓存到各个节点上，不需要用户手工部署。
Hadoop的分布式缓存并不是在内存中，而是在各节点的本地磁盘上。

按照可见性，资源分为PUBLIC，PRIVATE和APPLICATION

## 目录结构管理

NM上的Container在运行时也许会有大量的输出文件（比如Map任务的中间结果）要写到磁盘。为了避免干扰，通常会需要NM配置多个挂接到不同磁盘的目录。

NM在每个磁盘上为同作业创建不同的目录结构，以轮询方式把目录分配给不同的具体Container以避免冲突。运行结束后才统一清除中间数据。

日志目录管理也类似。

目录举例:
/hadoop/yarn/logs/tom/logs/applicaiton\_1344444444444\_0001/node0

## 状态机管理

NM 维护三类状态机:
-   Application:  跟踪应用程序在本节点的生命周期，是RM端Application信息的子集，两者生命周期一致
-   Container
-   LocalizedResource: 每个从HDFS下载的本地资源都需要一个状态机跟踪

## Container生命周期剖析

AM-&#x2014;startContainer(RPC)&#x2014;>NM:ContainerManagerImpl

三个阶段:

-   资源本地化，又包含应用程序初始化(准备服务，比如日志记录组件，资源状态追踪组件)和Container本地化(工
    作目录，从HDFS下载各类文件资源)
-   Container启动 ContainerExecutor  线程池中执行 创建Token文件和执行脚本
-   资源清理

## 资源隔离

两种资源隔离

-   内存资源有两种监控方案
    -   线程监控方案
    -   Cgroups
-   CPU资源就是采用Cgroups进行隔离

Cgroups:

-   限制进程组使用的资源量
-   进程组的优先级控制
-   记录进程组使用的资源量
-   进程组控制

术语

-   任务
-   控制组
-   层级
-   子系统

CPU子系统实际控制的是时间片，通过Linux CFS实现的。

创建子进程时，JVM采用了for+exec模型，进程创建后的复制行为会导致瞬间内存翻倍。因为这个关系，YARN采用了默认启动内存监视线程的方案（避免误杀，对进程还赋予年龄概念）。

CPU资源隔离：启用LinuxContainerExecutor。LinuxContainerExecutor的安全性的核心是以应用程序提交者的身份运行Container

注意Cgroups只能保证应用程序的CPU使用量的下限。

# 一些对比

另一种Hadoop生态圈中很流行的Mesos，在各方面都和Yarn很像。个人觉得没有优势。

Google的Omega是所谓的第三代资源管理系统，是另外一种思路。

三代如图所示：

<img style="mergin:5px;" src="/images//3gene.PNG">

第一代是集中式调度，是MRv1的样子，适合单一集群中单一类型的作业。管理比较粗糙；

第二代是YARN，Mesos这样的系统，有两级的资源管理，多用户多队列的细化管理方案，有比较丰富的资源调度算法；

第三代以Omega为例，一样还是为了支持大集群的多种计算框架的多类型作业，和第二代不同在于，第二代是所谓悲观锁，第三代是所谓乐观锁。悲观锁我的理解是系统倾向于相信资源是不够的，必须有大管家集中管理、控制，而乐观是系统的不同计算框架都同时直接看到资源的使用情况，并自己取用资源，对资源的占有是乐观态度。

很显然，YARN这样的系统，效率会较低，主控单元的单点故障对系统影响大，但是通过集中控制，能够做到比较公平，相对不容易饿死业务；而Omega需要做到资源使用对所有框架可见可写，资源申请和使用效率高，但是容易出现冲突（因为没有集中控制），且优先级低的业务更容易被饿死。
