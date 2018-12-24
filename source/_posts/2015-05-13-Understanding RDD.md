---
title: Spark：RDD理解
author: 梅溪
comments: true
layout: post
slug: 
tags : [大数据,Spark]
date: 2015-05-13  12:49:12+00:00
---


# Spark: 理解RDD

RDD是Spark系统的核心内容。在Spark系统中的位置如图所示。

<img style="mergin:5px;" src="/images/RDD.PNG">

## 概述

RDD全称是Resilient Distributed Datasets, 有人将之译为弹性分布式数据集。从字面上看，所谓Resilient，是指此数据集有分区，可以动态调整分区（也就是划分计算单元）；所谓Distributed，是指可以有多个计算单点协同计算，注重数据共享；所谓Datasets，是一个数据集合。
<!--more-->
RDD是MR模型的一种简单的扩展和延伸。RDD的设计者认为，不适合MapReduce的计算任务（例如，迭代，交互性和流查询）有一个共同点：在并行计算阶段之间要求能够高效地数据共享，这正是RDD试图改进的地方。

RDD的计算功能，核心理念是所谓[Lazy Computing](http://zh.wikipedia.org/zh/%E6%83%B0%E6%80%A7%E6%B1%82%E5%80%BC)，即延迟计算。

传统的提供容错性的方法就要么是在主机之间复制数据，要么对各主机的更新情况做日志记录。这两种方法对于数据密集型的任务来说代价很高，因为它们需要在带宽远低于内存的集群网络间拷贝大量的数据，同时还将产生大量的存储开销。RDD通过数据只读、粗粒度操作和记录依赖来保证容错。

综上，RDD的基本特征列举如下：

-   RDD数据本身是只读的；
-   RDD只支持粗粒度的操作: 一个操作应用到所有数据；
-   记录RDD上所做的转换操作，构建RDD的血统(lineage)，就可以有效地进行容错处理。一个Spark程序中，我们所
    用到的每一个RDD，丢失或者操作失败后都是可以重建的；
-   RDD是可分区的，RDD是分布式的。这两条是做分布式计算的最基本要求。

## 为什么数据是只读的

RDDS被设计为不可变的主要原因是为了跟踪不同版本的数据集的依赖关系依赖，并恢复依赖于旧版本数据集的状态。但是我觉得，做成数据可变也不是不能做到这一点，应该只是难度、复杂度和收效权衡后的结果。

Spark的论文承认不可变性和容错带来了一些开销。

但是基于下面两项技术，还是得到了不错的性能。

-   一个数据集的一些字段可能是一直不变的。每次迭代都是修改部分数据。只需要维护少量修改就可以了。
-   内部的数据结构的不可变性也可以充分利用指针挖掘性能。比如(Int, String)这样一个map，如果Int变化，
    而String不变，则后面的数据结构的String指针还可以指向前面的实际String，而不需要完全复制。

结论就是：数据只读不是一个分布式数据集的必需条件，只是这样更容易控制（不然就是一个随时可写的共享的可变状态系统）。而RDD不可写的实现目前看也还有不错的性能。Spark的研发人员还在进一步优化，以期得到接近可变状态系统的性能。

## 为什么只支持粗粒度的变换

RDD提供一种基于粗粒度变换（如map, filter, join）的接口，该接口会将相同的操作应用到多个数据集上。这使得他们可以通过记录用来创建数据集的变换（lineage），而不需存储真正的数据，进而达到高效的容错性。

只能基于粗粒度变换确实显得很局限。面对有大量的细粒度写操作的应用，RDD在使用上恐怕是受限制的。

但是这样也有很明显的好处：容错方面的效率得到了提升。如果有更细粒度的写操作，checkpoint的开销会变大，而且回滚操作的版本会太多，系统复杂度难以控制。

按我的理解，RDD比较适合对数据集中所有的元素做相同操作的应用。有大量细粒度更新的应用比如网络爬虫的增量存储就不太合适。RDD也并不适合实现一个分布式的key-value存储系统。不知道现在Spark在实际应用中，如何解决细粒度修改的问题。这点就留待后面再做深入研究了。

## 从基本抽象接口理解RDD兼对照MR

本章节从描述RDD所需要的抽象接口来说明RDD的特性。MR是指传统Hadoop Map/Reduce 式的计算模型。(把RDD和
M/R对比可能不太严谨，因为前者是一个数据集抽象，后者是一个计算框架，而且RDD也可以用作MR计算。权作参考吧)。

要表达RDD，所必需的抽象接口:

-   partition
-   preferredLocations(p)
-   dependencies()
-   compute(p, context)
-   partitioner()

### partition, partitioner, preferredLocations

partitions()返回RDD所有分区的数组。是分区列表；
partitioner()指定分区方法。是分区策略；
preferredLocations()根据数据的本地特性，列出分片p能够快速访问的节点。可以认为是分区位置；

分区决定计算粒度。每一个RDD分区的计算操作都在一个单独的任务中被执行。分区是为了更好地利用数据和计算的局部性原理。

RDD的思想是：移动数据不如移动计算。要做到这一点就要有接口能够得到数据位置的相关信息以便调度程序据此分配任务。

我们对比MR，其实分区的思想也是有的:

-   MR在进行Map处理时已经考虑了距离输入分片最近的tasktracker;
-   对Map处理，MR可以通过用户自实现的partitioner程序来优化Map结果的分派。但是只能控制分区的数目，可以
    理解为调整Reduce程序的并发程度，不涉及计算和位置的关联性;
-   对于Reduce 任务，JobTracker是随机选择一个，并不考虑数据本地化。可以说Reduce 任务跨网络的传输是更常
    见的。

对比之下，RDD的思路是提供更显式更严格的分区功能。用户对分区可以有更细的控制。比如用户可以在算法上控制把有任务相关性的多个RDD按照相同的分区方式进行分区。

Spark提供了更精细的分区控制。当然，具体计算的本地效果如何还要取决于数据的分布，数量，计算过程和算法。
Spark或者说RDD并不是银弹，不能解决所有问题，而只是一种机制，提供了更多的可能性。

### dependencies


RDD每一步转换操作产生新的RDD，形成流水线般的前后依赖。RDD会把所有的依赖作为一张图记录。

所谓Narrow和Wide之分如图所示：

<img style="mergin:5px;" src="/images/narrow.PNG">

为什么要特别强调宽和窄？

-   窄依赖更容易作为连贯计算步骤的一部分，且更易于计算失败后恢复RDD。
-   宽依赖则因为出现了数据在分区的交换，无法避免和MR相同的shuffle过程。这一点也就涉及了RDD的计算阶段的划分。

对比MR，MR存在事实上的依赖: M->Shuffle->R，但没有对依赖的表达，或者说没有太多对依赖的利用。

Spark的依赖，我觉得最重要的意义就是提供了一种便于溯源和容错恢复的记录。

关于容错这一点，M/R任务失败就是重新执行M或R。RDD因为有记录所谓血统(lineage)，相当于包含了RDD转换所需要的逻辑，重新执行特定的步骤可以恢复丢失的分区。

Spark的容错，基本思路是内存中记录更新点，结合少量checkpoint保存在可靠介质。

### compute

非常有趣。compute 函数只返回相应分区数据的迭代器，只有最终实例化时才能显示最终结果。

使用迭代器而不是每一步骤都立刻计算产生结果，这一点有利于节约内存。这是RDD可以利用内存进行大规模计算的前提。这也就达成了所谓的Lazy Computing。

熟悉Python的同学应该很容易理解，直接计算和RDD的compute的效果，就类似于这两种写法的差别(代码和RDD无关，只是为了说明效果):


    #对比: list comprehension和generator
    #立刻计算
    >>> [x*2 for x in range(1,10)]
    [2, 4, 6, 8, 10, 12, 14, 16, 18]
    >>> 
    >>> 
    #返回生成器
    >>> (x*2 for x in range(1,10))
    <generator object <genexpr> at 0x00000000021D86C0>



再举个真正的scalar操作RDD的例子:



    //RDD: lines, errors
    lines = spark.textFile("hdfs://. . .“)
    errors = lines.filter(_.startsWith(MERROR"))
    errors.persist()



-   第一行，lines是从HDFS读取文件得到的RDD；
-   第二行，errors是执行了过滤，找到文件中的特定字符开始的行；
-   第三行，errors执行了持久化。

此过程中,lines这个RDD并不需要加载到内存中(考虑到实际errors可能只是一小部分，这样做是值得的)。而对于errors，在对其有任何具体行动之前也不会做任何实际计算。

对比MR：MR没有提供惰性计算的能力，计算的表达也比较太乏力。所谓Map，Reduce只是一个比较粗线条的框架描述。

相比之下，RDD提供了丰富的基于数据集的运算操作，少量代码可以完成复杂计算。而且RDD在底层用迭代器来表达计算，在大量迭代计算的场景对MR有压倒性的优势(在实际宣传时，Spark也往往正是标榜在迭代计算上比传统
Hadoop领先多少倍)。

### 综述

从基本API的设计来看:

-   RDD更注重计算的本地化，加强分区控制；
-   RDD更注意计算步骤的依赖关系，更有利于追溯，包括容错；
-   RDD的计算在表达力上比M，R更强，代码写起来更简洁（我感觉不一定简单）;
-   RDD的计算有更强大的对迭代的支持；
-   RDD的计算结果可以指定cache到内存，在介质上比较M,R更快。

## RDD支持的操作

下面简单列举RDD支持的操作。

### 基本操作: 创建RDD

-   sc.makeRDD
-   parallelize

### 基本操作: 转换

基本转换操作:

-   map             1 -> 1
-   distinct        返回所有不一样的元素
-   flatMap         1 -> N
-   repartition     重新分区
-   coalesce(较复杂, 考虑shuffle 参数， 没太明白)
-   randomSplit    给定加权，随机切分RDD并返回
-   glom           把所有元素作为一个Array返回
-   union          将两个RDD的数据合并，返回并集但不去重
-   intersection   交集
-   subtract      减去另一个RDD的元素
-   mapPartitions  按分区map
-   mapPartitionsWithIndex
-   zip            把2个RDD按照k,v 结合
-   zipPartitions

### 基本操作: 键值RDD转换

-   partitionBy
-   mapValues
-   flatMapValues
-   combineByKey
-   foldByKey
-   reduceByKey
-   groupByKey
-   cogroup
-   join
-   leftOuterJoin
-   rightOuterJoin
-   subtractByKey

Spark在计算过程中内部生成的RDD多于定义的RDD。这应该也很容易理解: 一些复杂的计算操作是基本操作的复合。

### 基本操作: 控制操作

在做cache这样的动作之前，RDD是没有“实体化”的。

-   cache
-   persist    cache 和persist默认都是存到内存。persist还可以指定其他存储介质
-   checkpoint  持久化到HDFS，且作为还原点。过多的依赖对系统资源占用高，容错重算成本也高。少量的

checkpoint还有必要的。

### 基本操作: 行动操作

前面提过compute过程。无论执行了多少次转换操作，RDD都不会真正执行运算，只有当action操作被执行时，运算才会触发。从字面上看都还比较容易理解，不再多说。

-   first
-   count
-   reduce
-   collect
-   take
-   top
-   takeOrdered
-   aggregate
-   fold
-   lookup

### 存储行动操作

可以把数据存入本地或HadoopFile。

## 总结

Spark的兴起，我觉得除了强调内存计算，其实更重要是RDD逻辑表达上的给力。提供通用的计算元语而不是具体的计算模型，可以更好地据此在更上层根据需要提供Streming，SQL，Graph等差别化的服务，让更多的开发者参与进来，才能形成繁荣的生态圈。

Spark的RDD的设计，Scalar语言的选取，都体现了函数式编程的思想。个人初步的感受，是实际的编码和操作并不简单。哪些RDD需要在内存记录，哪些RDD不用记录全靠恢复，哪些时机需保存checkpoint，分区策略……需要考虑的很多。

Spark具体如何应用到各种数据处理场景，还需要继续深入学习。

初学Spark，只是一些粗浅见识，不当之处还请指出。


