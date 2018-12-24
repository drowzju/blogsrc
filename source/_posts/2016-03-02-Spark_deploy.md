---
title: Spark on Ubuntu
author: 梅溪
comments: true
layout: post
slug: 
tags : [大数据]
date: 2016-03-02 15:11:46
---
# Spark deploy

前一篇文中我们部署了一个具备HDFS HA和YARN HA的Hadoop小集群。

这次我们在这套环境上继续部署一套Spark引擎。部署过程相当简单。
<!--more-->
## 安装java环境和scala
下载安装略

## 下载，解压Spark
我用的是1.5.2版本，下载解压过程略。

## 环境变量修改

我们修改/etc/profile，里面其实就是指定java位置，scala位置，Hadoop和Spark位置，以及把相关bin加入PATH。

```
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/
export SCALA_HOME=/usr/share/java
export HADOOP_HOME=/home/ubuntu/hadoop-2.7.1
export SPARK_HOME=/home/ubuntu/spark-1.5.2-bin-hadoop2.6
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$HADOOP_HOME/bin:$SCALA_HOME/bin:$SPARK_HOME/bin:$PATH
```

修改spark/conf/spark-env.sh。这里注意指定了Spark的Master角色，WORKER的内存量(1024是比较小了),Hadoop位置。

```
export SCALA_HOME=/usr/share/java
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/
export SPARK_MASTER_IP=192.168.103.47
export SPARK_WORKER_MEMORY=1024m
export HADOOP_CONF_DIR=/home/ubuntu/hadoop-2.7.1/etc/hadoop
```


## 编辑slaves

Spark的这个slaves指的是可以被Master管理的机器。

看过前一篇文的同学都知道我是有4个计算节点，都加入spark/conf/slaves。

```
hulk.dtdream
captain.dtdream
ironman.dtdream
thor.dtdream
```


## 启动并验证

启动Master和所有Slave。

```

./sbin/startall.sh

```

看版本信息，简单验证

```

spark-shell --version

```

跑个蒙特卡洛逼近Pi的程序，简单验证。

```
run-example org.apache.spark.examples.SparkPi 10

```


向YARN提交个作业，验证Spark on YARN:

```

spark-submit --master yarn-cluster --class org.apache.spark.examples.SparkLR --name SparkLR ~/spark-1.5.2-bin-hadoop2.6/lib/spark-examples-1.5.2-hadoop2.6.0.jar

```

登录http://你的master ip:8080，可以看看Spark的Web UI。


## 配置和调优(To be continued...)

有时间再补充。
