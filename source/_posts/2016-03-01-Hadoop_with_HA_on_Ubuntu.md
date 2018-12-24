---
title: Hadoop HA on Ubuntu
author: 梅溪
comments: true
layout: post
slug: 
tags : [大数据]
date: 2016-03-01 19:11:46
---
# 从0开始在Ubuntu搭建高可靠Hadoop环境

本文用于指导基于Ubuntu搭建大数据的一个小型Hadoop实验环境，此环境具备文件系统和资源调度系统的HA。

## 环境和目标

### 操作系统我使用的是Ubuntu 14.04.1，其Linux内核版本是3.16.0-30-generic。

<!--more-->
### 安装组件

- 我们要安装Hadoop2.7.1，提供HDFS/YARN/MapReduce，分别是分布式文件系统，分布式资源调度系统和离线计算框架；
- 安装Zookeeper3.4.7，提供分布式协同，统一命名服务；
- 安装Spark1.5.2，提供内存计算引擎;
- 安装Hive1.2.1，基于Hadoop提供SQL能力；
- 安装Docker环境，并和YARN结合。

本文先介绍Hadoop和Zookeeper的相关内容，其余内容后续再做展开。

### 硬件: 主控和计算分离

我这里是两种机型，一种作为控制节点，一种作为计算节点。计算节点的CPU，内存和磁盘资源都更加充足。

一般说来，在物理上划分控制和计算，更容易做到资源的管控和计算的稳定，以及成本的控制。

(但这并不是必须的要求，全对称的机型看起来更美。)

下面是我的具体硬件环境说明，我想在当下(2015年)作为一个实验环境来说算是很不错的配置了。

3台主控节点

- 内存: 96GB（6*16GB）
- CPU: E5-2650 V2 8核 * 2
- 硬盘: 600G * 4  Raid 

3台主控节点，我分别命名为wonderwoman,superman和batman 

4台计算和存储节点

- 内存: 128GB(8*16G)
- CPU: E5-2650 V2 8核 * 2
- 硬盘: 4T * 12

4台计算节点，我分别命名为hulk, captain, ironman 和 thor。

对，我这段时间在看美漫。


### 文件系统和调度系统高可靠

文件系统和调度系统做到高可靠，避免因单点故障而导致集群无法工作。


## 系统准备

### 安装OS
略。

### 用户不建议全以root为用户进行操作。

作为非root用户，在一些关键操作也需要能够sudo。以我的环境为例，我是使用ubuntu这个用户，需要增加一个/etc/sudoers.d/nopasswd4sudo这样的文件，内容为:
 
```
%ubuntu ALL=(ALL) NOPASSWD: ALL
```

### 打通SSH免秘登录

所有讲分布式集群部署都需要这一步，不再赘述了。效果就是要求各节点彼此间能够免密码ssh登录。

### 安装pssh

pssh是一个很好用的工具，便于你在多台机器上同时执行相同命令。下文中很多操作比如把文件拷贝到多台机器或者全部执行某命令都是靠它。


### 格式化磁盘并挂载

我的计算节点有多个数据盘(除了系统盘外还有11块盘之多！)，下面是我写的一个格式化磁盘并挂载到制定目录的一个脚本，用于把多块磁盘挂载到指定目录，并把配置写入/etc/fstab以保证重启后有效。


```
#!/bin/bash
#!useage: fmtandm.sh dev dir
if [ $# != 2 ] ; then
    echo "USAGE: $0 DEV DIR"
    echo " e.g.: $0 sdb data0"
    exit 1;
fi 

#创建目录并挂载。免确认。
mkdir /$2
(echo "y"
sleep 5
echo 
sleep 2  ) | mkfs -t ext4 /dev/$1
mount -t ext4 /dev/$1 /$2
x=`blkid /dev/$1 |  sed -r "s/.*UUID=\"([^ ]*)\".*/\1/"`
echo UUID=$x /$2 ext4 errors=remount-ro 0 1>>/etc/fstab
```



## 安装Zookeeper和Hadoop

### Java环境安装

修改/etc/profile 之类，略。

### 高可靠说明

#### HDFS HA

HDFS的HA有两个问题。

- 一个问题是解决NameNode的主备状态，一个ActiveNameNode，一个StandbyNameNode，且Active故障时Standby要能立刻接管。

我们通过ZKFC(DFSZKFailoverController)，利用Zookeeper集群来决定NameNode的主备状态确认和切换。

- 另一个问题是主备NameNode的数据的同步。
 
目前在小集群中一个可用性较强的方案是通过使用QJM(Qurom Journal Manager)来实现。QJM也是一个Paxos类型的协议，通过2N+1个所谓JournalNode记录NameNode的edit log。通过QJM，Active NameNode的所有修改都对Standby NameNode可见(Standby要将修改apply到自己的namespace)，而一旦Active失效，Standby在接管之前，一定要保证读完所有JournalNode的修改记录。


我在三台主控节点中，选择superman 为Active NameNode，选择wonderwoman为Standby NameNode。

选择三台计算节点作为JournalNode。也可以选择所有4台计算节点为JournalNode，不过我们都知道Paxos是要求2N+1，这样不太合适。

相关配置参见后续具体步骤。

#### YARN HA

YARN的资源主控单元叫Resource Manger，HA是要解决Resource Manager的单点问题。

YARN的HA无论在功能上还是配置上其实都更简单一些(不过因为YARN出现比HDFS晚，所以HA方案出现也晚)。

YARN的客户端，AppMaster以及NodeManager都会以轮询方式查找Resource Manager。YARN的HA也是要保证Resource Manager只有一个主控，且主控故障时另一个能够响应。

相关配置参见后续具体配置。


### 安装Zookeeper

- 下载包(略)
- 解压(其实也可以略)并拷贝到各个节点(略)

```
tar -zxvf /tmp/zookeeper-3.4.7.tar.gz -C ~/
```

- 修改配置conf/zoo.cfg。dataDir是你的数据存放目录，保证对应节点有此目录。server.x是各个节点。我是选择了3个主控节点组成zookeeper集群。


```
dataDir=/home/ubuntu/zookeeper-3.4.7/data

server.1=wonderwoman.dtdream:2888:3888
server.2=superman.dtdream:2888:3888
server.3=batman.dtdream:2888:3888
```

- 启动，iplist_dc是我准备好的ip列表，pssh不解释

```
pssh -i -h ~/iplist_dc  '~/zookeeper-3.4.7/bin/zkServer.sh start'
```

### 安装Hadoop

如无特殊说明，涉及的配置文件都在hadoop-2.7.1/etc/hadoop下面。

- 下载，解压，拷贝(略)
- 修改各个节点的/etc/profile，增加hadoop dir

```
export HADOOP_HOME=/home/ubuntu/hadoop-2.7.1
```
- 修改hadoop-2.7.1/etc/hadoop/hadoop-env.sh，增加JAVA_HOME

```
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/
```
- 保证我们使用的用户在各个节点的数据目录都有访问权限(iplist_dc和iplist_all 是我准备的节点IP列表文件)

```
pssh -i -h ~/iplist_dc 'sudo mkdir /data0'
pssh -i -h ~/iplist_all 'sudo chown ubuntu:ubuntu /data*'
```

#### 使用多块硬盘

我们的计算节点机型有多块硬盘(一块系统盘，11块用作HDFS数据盘)，相应的，计算节点上HDFS的配置文件(hadoop-2.7.1/etc/hadoop/hdfs-site.xml)中要如此声明。

```
        <!-- 配置数据目录 -->
        <property>
                <name>dfs.data.dir</name>
                <value>/data0,/data1,/data2,/data3,/data4,/data5,/data6,/data7,/data8,/data9, /data10</value>
        </property>
```

#### 配置core-site.xml

集群所有节点同配置。


```
<configuration>
    <!-- 指定hdfs的nameservice为ns1。ns1具体是定位为哪个节点见hdfs-site.xml -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://ns1</value>
    </property>
    <!-- 指定hadoop临时目录。临时目录默认情况下也会作为元数据目录，除非显式指定dfs.name.dir -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data0/tmp</value>
    </property>
    <!-- 指定zookeeper地址 -->
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>wonderwoman.dtdream:2181,superman.dtdream:2181,batman.dtdream:2181</value>
    </property>
</configuration>
```


#### 配置hdfs-site.xml

同样，集群所有节点相同配置。

先看ns相关定义。注意这里我们ns1 对应有两个物理节点nn1和nn2。

```
    <!--指定hdfs的nameservice为ns1，需要和core-site.xml中的保持一致 -->
    <property>
        <name>dfs.nameservices</name>
        <value>ns1</value>
    </property>
    <!-- ns1下面有两个NameNode，分别是nn1，nn2 -->
    <property>
        <name>dfs.ha.namenodes.ns1</name>
        <value>nn1,nn2</value>
     </property>
    <!-- nn1的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.ns1.nn1</name>
        <value>superman.dtdream:9000</value>
    </property>
    <!-- nn1的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.ns1.nn1</name>
        <value>superman.dtdream:50070</value>
    </property>
    <!-- nn2的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.ns1.nn2</name>
        <value>wonderwoman.dtdream:9000</value>
    </property>
    <!-- nn2的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.ns1.nn2</name>
        <value>wonderwoman.dtdream:50070</value>
    </property>
```


JournalNode相关配置。还记得前面讲的HDFS HA么？这里我们选择了hulk,captain和ironman做JournalNode

```
    <!-- 指定NameNode的元数据在JournalNode上的存放位置 -->
    <property>
        <name>dfs.namenode.shared.edits.dir</name>        
        <value>qjournal://hulk.dtdream:8485;captain.dtdream:8485;ironman.dtdream:8485/ns1</value>
    </property>
    <!-- 指JournalNode在本地磁盘存放数据的位置 -->
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/home/ubuntu/hadoop-2.7.1/journal</value>
    </property>
```

Failover切换配置


```
    <!-- 开启NameNode失败自动切换 -->
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <!-- 配置失败自动切换实现方式 -->
    <property>
        <name>dfs.client.failover.proxy.provider.ns1</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
```

#### MapReduce引擎配置

同样是所有节点。

这里只是指定了YARN作为其调度框架。

```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```


#### YARN配置

打开HA，以及Zk相关配置。

```

    <!--开启resource manager HA,默认为false-->
    <property>   
    <name>yarn.resourcemanager.ha.enabled</name>  
    <value>true</value>
    </property>
    <property>  
    <name>ha.zookeeper.quorum</name> 
    <value>wonderwoman.dtdream:2181,superman.dtdream:2181,batman.dtdream:2181
    </value>
    </property>
    <!--配置与zookeeper的连接地址-->
    <property> 
    <name>yarn.resourcemanager.zk-state-store.address</name>  
    <value>wonderwoman.dtdream:2181,superman.dtdream:2181,batman.dtdream:2181
    </value>
    </property>
    <property>  
    <name>yarn.resourcemanager.store.class</name>  
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore
    </value>
    </property>
    <property>  
    <name>yarn.resourcemanager.zk-address</name>  
    <value>wonderwoman.dtdream:2181,superman.dtdream:2181,batman.dtdream:2181
    </value>
    </property>

```

RM相关配置 

```

    <!--配置resource manager -->
    <property>  
    <name>yarn.resourcemanager.ha.rm-ids</name>  
    <value>rm1,rm2</value>
    </property>
    <property>  
    <name>yarn.resourcemanager.hostname.rm1</name> 
    <value>batman.dtdream</value>
    </property>         
    <property>   
    <name>yarn.resourcemanager.hostname.rm2</name> 
    <value>wonderwoman.dtdream</value>
    </property>
    <!--在batman上配置rm1,在wonderwoman上配置rm2-->
    <property> 
    <name>yarn.resourcemanager.ha.id</name> 
    <value>rm1</value>
    <description>If we want to launch more than one RM in single node, we need this
    configuration</description>
    </property>
    <!--配置rm1-->
    <property>
    <name>yarn.resourcemanager.address.rm1</name>
    <value>batman.dtdream:8132</value>
    </property>
    <property> 
    <name>yarn.resourcemanager.scheduler.address.rm1</name> 
    <value>batman.dtdream:8130</value>
    </property>
    <property>  
    <name>yarn.resourcemanager.webapp.address.rm1</name> 
    <value>batman.dtdream:8188</value>
    </property>
    <property> 
    <name>yarn.resourcemanager.resource-tracker.address.rm1</name> 
    <value>batman.dtdream:8131</value>
    </property>
    <property>  
    <name>yarn.resourcemanager.admin.address.rm1</name>
    <value>batman.dtdream:8033</value>
    </property>
    <property> 
    <name>yarn.resourcemanager.ha.admin.address.rm1</name> 
    <value>batman.dtdream:23142</value>
    </property>
    <!--配置rm2--><property> 
    <name>yarn.resourcemanager.address.rm2</name>
    <value>wonderwoman.dtdream:8132</value>
    </property>
    <property>  
    <name>yarn.resourcemanager.scheduler.address.rm2</name> 
    <value>wonderwoman.dtdream:8130</value>
    </property>
    <property> 
    <name>yarn.resourcemanager.webapp.address.rm2</name> 
    <value>wonderwoman.dtdream:8188</value>
    </property>
    <property>  
    <name>yarn.resourcemanager.resource-tracker.address.rm2</name> 
    <value>wonderwoman.dtdream:8131</value>
    </property>
    <property>  
    <name>yarn.resourcemanager.admin.address.rm2</name>
    <value>wonderwoman.dtdream:8033</value>
    </property>
    <property> 
    <name>yarn.resourcemanager.ha.admin.address.rm2</name>
    <value>wonderwoman.dtdream:23142</value>
    </property>

```

故障恢复、处理相关

```

    <!--开启故障自动切换-->
    <property>  
    <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>  
    <value>true</value>
    </property>
    <!--开启自动恢复功能-->
    <property> 
    <name>yarn.resourcemanager.recovery.enabled</name>  
    <value>true</value>
    </property>
    <!--schelduler失联等待连接时间-->
    <property>  
    <name>yarn.app.mapreduce.am.scheduler.connection.wait.interval-ms</name> 
    <value>5000</value>
    </property>
    <!--rm失联后重新链接的时间-->
    <property>  
    <name>yarn.resourcemanager.connect.retry-interval.ms</name>   
    <value>2000</value>
    </property>
    <!--故障处理类-->
    <property> 
    <name>yarn.client.failover-proxy-provider</name>
    <value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
    </property>
    <property>   
    <name>yarn.resourcemanager.ha.automatic-failover.zk-base-path</name>   
    <value>/yarn-leader-election</value>   
    <description>Optional setting. The default value is /yarn-leader-election</description>
    </property>

```



### 启动和初始化

前面提过了Zookeeper的启动。


启动所有journal node。注意这里是daemons

```
ubuntu@superman:~/hadoop-2.7.1$ sbin/hadoop-daemons.sh start journalnode 
```


在一个Namenode上执行HDFS 的NameNode格式化: 

```
hdfs namenode -format 
```

手动拷贝hadoop临时目录内容(参见core-site.xml)到另一个NameNode上:

```
scp -r /data0/tmp/ wonderwoman.dtdream:/data0/
```

HDFS 涉及的ZKFC(ZooKeeperFailoverController)数据初始化:

```
hdfs zkfc -formatZK
```


HDFS启动

```
ubuntu@superman:~/hadoop-2.7.1$ sbin/start-dfs.sh 
```

应该能看到一系列namenode，datanode和journalnode启动的输出信息，还可以看到ZKFC的信息，说明我们的NameNode HA已经生效:

```
Starting ZK Failover Controllers on NN hosts [superman.dtdream wonderwoman.dtdream]
superman.dtdream: starting zkfc, logging to /home/ubuntu/hadoop-2.7.1/logs/hadoop-ubuntu-zkfc-superman.dtdream.out
wonderwoman.dtdream: starting zkfc, logging to
/home/ubuntu/hadoop-2.7.1/logs/hadoop-ubuntu-zkfc-wonderwoman.dtdream.out
```

YARN 启动

```
ubuntu@batman:~/hadoop-2.7.1$ sbin/start-yarn.sh 
starting yarn daemons
starting resourcemanager, logging to /home/ubuntu/hadoop-2.7.1/logs/yarn-ubuntu-resourcemanager-batman.dtdream.out
ironman.dtdream: starting nodemanager, logging to /home/ubuntu/hadoop-2.7.1/logs/yarn-ubuntu-nodemanager-ironman.dtdream.out
captain.dtdream: starting nodemanager, logging to /home/ubuntu/hadoop-2.7.1/logs/yarn-ubuntu-nodemanager-captain.dtdream.out
thor.dtdream: starting nodemanager, logging to /home/ubuntu/hadoop-2.7.1/logs/yarn-ubuntu-nodemanager-thor.dtdream.out
hulk.dtdream: starting nodemanager, logging to
/home/ubuntu/hadoop-2.7.1/logs/yarn-ubuntu-nodemanager-hulk.dtdream.out
```

在作为standby RM 的wonderwoman上也是start-yarn.sh，然后

检查YARN的两个Resource Manager的状态，应该看到一主一备:

```
ubuntu@batman:~/hadoop-2.7.1$ yarn rmadmin -getServiceState rm1
active

ubuntu@batman:~/hadoop-2.7.1/etc/hadoop$ yarn rmadmin -getServiceState rm2
standby
```


至此，我们的一个具备了文件系统和资源调度系统高可靠的Hadoop小集群已经Ready。后续文章中我们会在其上继续增加Spark的部署、Hive的部署，以及Docker环境的集成。

