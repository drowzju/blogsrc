---
title: 基于Spark 的企业级大数据平台建设实践（一）
author: 梅溪
comments: true
layout: post
slug: 
tags : [大数据]
date: 2017-03-29 09:55:00
---



本文简述我司大数据产品的早期建设过程。我们基于Hadoop和Spark，提供了和Hive相同界面的SQL引擎，完整的数据交换链路，以及带图形化的大数据工作平台。

本篇是系列的第一篇——离线计算系统的建立，重点放在SQL上。



## 需求和场景

我们是一家不大不小的创业公司，缺少互联网大厂的人力物力资源和业务场景，在技术上要求有如下特点。

- 满足大数据应用的需求，可能会来自于不同行业，定制化程度高；
- 平台开放性，更容易形成合作。要同时考虑开源生态圈和不同行业ISV；
- 小型化，弹性扩容；
- 便于输出，向非技术背景客户提供助力。特别要求部署简单，使用门槛尽可能低；


<!--more-->

## 技术选型


我们的目标是要用较少的人力(关键词：初创，小厂)支撑一个相对复杂的大数据分布式计算平台，所以选型上更多注意对技术成本的控制。

### 数据来源和数据存储


Spark的一大优势是在设计上方便加入新的数据源，比如直接导入一个Json，比如直接取Mysql数据作为RDD计算，比如自己编码实现HBase的DataSource。



我们暂时最重要需求还是数仓，体现为Hive的管理表，所以我们的选择中规中矩:

- 分布式文件系统HDFS
- 目前以parquet格式为缺省格式，高压缩比，列式存储



后续的数据源还是以需求驱动，必然又是差异化、定制化的场景。

### 计算引擎


太小众的就不考虑了。

- MapReduce   成熟稳定，表现力差，写屏障多，性能差
- Tez         基于MapReduce，利用DAG消除一定写屏障，相对还是小众
- Spark       迭代计算，写屏障少,性能好，接口表现力强，对内存依赖大，稳定性相对差，社区最活跃
- Flink       大有后来居上的趋势，理念比较先进。


选择Spark。考虑到大厂们都开始把MapReduce的程序往RDD，DF去迁移，Spark越来越多地应用于生产实践，这一点其实也没什么争议了。

Flink有越来越多的人推崇，但是毕竟还比较新，我们还是再等等。
(这里需要多关注一下Apache Beam的发展，它本身不是计算引擎而是大数据的处理“标准”——Google总是更倾向于引领标准的建立，Flink是对Beam支持最好的。)



### 资源调度系统

- Mesos               在业界达到生产环境实施标准，更稳定
- YARN                社区相对活跃，同属Hadoop的生态圈，为大数据而生，更易用
- Spark StandAlone    云上所得到的Spark集群的常见调度方式，毕竟对用户所见最简单

目前是选择了YARN。没有选择使用Spark Standalone(对于小集群，单一应用更合适)，是因为目前看我们暂时没有一个可靠的云(我们的客户一般不选择公有云，而我们暂时也无力同时投入私有云的建设和大数据系统的融合)，且还是有多种应用复用同一集群（Hive/Spark，MapReduce……）、有更复杂隔离需求的可能。

从业界的口碑看可能选择Mesos会更好。目前YARN满足使用，暂时还没有投入更多分析。







### SQL界面和接入方式


Spark提倡用一种代码中嵌入SQL的方式，对程序员更友好，更方便。

``` scala
    val sqlContext = new org.apache.spark.sql.SQLContext(sc)
    val df = sqlContext.read.json("../examples/src/main/resources/people.parquet")
    //查看所有数据：
    df.show()
    //查看表结构：
    df.printSchema()
    //只看name列：
    df.select("name").show()
    //对数据运算：
    df.select(df("name"), df("age") + 1).show()
    //过滤数据：
    df.filter(df("age") > 21).show()
    //分组统计：
    df.groupBy("age").count().show()
```    

这种方式非常灵活，但作为数仓应用，我们更希望提供的是一个SQL界面。


所以我们优先选择提供一个Thrift Server方式，需要用户交互式输出查询，也可以编程通过jdbc接口访问。


在明确了SQL界面，接入方式，元数据形式之后，这里其实还有一个选择项：是Hive on Spark，还是Spark with HiveFace？

这是一个艰难的选择，Hive作为老牌大数据SQL系统，对业务系统的很多细节支持更成熟。

我们暂时选择了Spark with HiveFace，出发点是更看好Spark社区的发展速度，同时也是减少对Hive组件的依赖。


### 数据同步


考虑以下几种方式的数据同步。

- 既有数据库数据导入导出
- 流式数据导入
- 其他






#### DB接入

主流的做法是使用Hive Insert/Select，我们的实现选择采用HDFS读写接口加Parquet插件。

主流的做法更加简单，没有开发工作量，但是导入性能差，且不可避免需要占用计算资源。

我们的实现方式基本上只有网络和磁盘IO，导入速度是前者的10倍。

这里有很多的坑，特别是DB->转换工具(Kettle)->Parquet File->Spark Catalyst Row  这个转换过程中涉及到最多四种类型的对应关系，教育了我们开源系统的整合之难。我们后续会对这块进行比较大的系统重构。

#### 流式数据

基于开源软件[confluent connector ](https://www.confluent.io/product/connectors/)做封装。
有空我再写一篇blog详细介绍这个子系统。

*DATABUS组件说明*

- Kafka：数据存储组件，提供高可靠，大吞吐，高并发的分布式存储；
- Schema Registry：schema服务组件，提供schema的管理，获取服务；
- Kafka connect：数据导出组件，将kafka中数据导出至HDFS，并建立HIVE和HDFS的关联关系。


![databus特性](/images/databus.png)

*基本流程*

- 数据汇集在beaver处；
- beaver调用databus相关接口，从schema registry获取schema信息，序列化数据，并发送至kafka；
- kafka connect 从kafka中读取数据，从schema registry中获取schema信息，反序列化数据；
- kafka connect 根据数据内容，通过hive metastore建立相关表和分区，并设定分区和hdfs目录的对应关系；
- kafka connect 将数据以parquet格式写入相应hdfs目录；





#### 其他

我们还提供了Restful API的方式进行数据的同步，便于用户自编程交换想要的数据。




### 元数据


Spark的元数据默认是以derby文件形式存在本地磁盘。这是一种比较原始的方式。

我们以MySQL做存储，对外接口和Hive 1.2.1兼容。

元数据的高可靠就由MySQL HA来保证。



### 非结构化数据

其实考虑非结构化数据之前，先要考虑的是数据源问题。


这个也必然是根据客户需求做定制化的地方。目前看Spark在计算和数据来源之间是有很好的解耦的，可以留待后续需求驱动。




## 架构一览


按照之前选型的结果，目前的一个架构

![特性架构](/images/emr-arch.png)



在某项目中实施时的业务架构(原谅我的马赛克)


![业务架构](/images/bigdata-scene.png)









## 引擎增强

简单列举一些我们趟过的坑。

### 解决UDF持久化问题

Spark 1.5, 1.6版本的UDF持久化都有问题，包括存入Meta DB和从其中读取都有问题。深度探究这个问题，从代码逻辑看，是Spark对待Hive Meta的态度相对谨慎，考虑了Hive Meta可能对接不同版本，所以实现并不完整（因为担心有问题，所以干脆不实现也实在是过于保守）。



我们的Spark和Hive Meta版本都固化，所以没有版本切换的负担，自行解决了这个问题。

### add UDF jar on HDFS

除了UDF持久化之外，UDF所涉及JAR包的持久化也需要考虑。放在分布式文件系统是比较简单的保证可靠性的持久化方式。

我们添加了spark add jar对HDFS的支持。并且考虑了关键服务比如STS重启后能够自动把相关的JAR重新加载起来。

### 添加udf时class not found引发异常

Hive 代码的bug，关闭classloader导致父classloader中持有已close的entry，引发后续异常，甚至无法新建thrift连接。

我们的STS作为常驻服务存在，这个bug比较恶劣。我们对之进行了修复。

### 支持delete jar

开源实现没有delete逻辑，导致udf的调试效率很低。

我们增加了delete jar流程。

### 解决一些内存异常

太多不计。

### 增加kill job rest api接口


为了运维需求，增加了spark的rest api，可以直接kill特定作业。

### 用户UDF python扩展


Hive原先所涉及的python用法，是只有所谓transform，原理是把执行过程中的输出重定向到python 脚本中计算完再返回，有点像是管道。

而Spark虽然有PySpark，相对灵活，但是是另外的命令行界面，并不能通过thrift server交互使用。


我们通过定义Java 容器和Jython组件，提供了Python UDF支持，并提供了在UDF中针对特定Hive DB去get resource的能力。




### 数据生命周期管理


实现Hive LifeCycle语法，在Hive/Spark中使用。可以针对表配置生命周期。

需要启动一个crontab任务定期扫描表，对生命周期达到的表和分区进行老化。

### 解决driver 内存泄露

Spark session 涉及的HiveConf泄露，是比较明显的bug。奇怪开源没有解决。不同版本的现象亦有不同。

我们在session退出时clean掉对应的内存。

### 高可靠


各个组件

- HDFS           基于ZKFC 做HA
- YARN           基于Zk做HA
- Kafka/DataBus  本身是集群，基于Zk
- Thrift server  目前单点。后续考虑容器化通过K8S进行管理。
- MySQL          既有的mysql-ha

### 自动化部署

通过Ambari实现。

### 运维系统

基于Ambari做扩展。增加了服务自检，集群时钟管理，日志自动搜集汇总等大量功能。






### 文件碎片优化


我们发现我们使用的Spark 1.6.1在读取Parquet Relation时对于小文件还是一个文件一个Task，大量碎文件就会带来大量的Task，比较浪费。对于STS、计算资源来讲都是不小的负担。

我们从两个角度来优化这个问题:

- 引入[Adaptive Execution](https://issues.apache.org/jira/browse/SPARK-9850)，减少shuffle过程中生成的文件数目；
- 对于原始数据源，修改代码，能够对比较小的文件做merge，合成一个partition进行处理。


### 优化Executor 内存overhead问题


这个问题可以参见[这篇博客](http://www.uml.org.cn/itnews/2016050308.asp) 其原因是Spark对于堆外内存的用量不明晰，导致被YARN杀死，是非常坑的。我参考了这篇博客提供的思路进行了修改。

### On Yarn长服务定义


我们也提到了，Spark 应用作为一个长服务是有一些挑战的，我们增加了一些自定义的配置项，声明为长服务，在容错方面做一些代码上的修改，以利长期稳定运行。






### 安全和隔离

- 安全功能需要继续增强
- 资源隔离。需要cgroup/yarn的配置。
- 数据隔离。更丰富的数据授权和访问控制机制。













