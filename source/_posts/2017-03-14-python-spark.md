---
title: Python On Spark
author: 梅溪
comments: true
layout: post
slug: 
categories: [大数据]
tags : [Spark, python]
date: 2017-03-14 13:02:16
---




## Python: 数据分析人员的热门语言

如果你关心数据圈子，你会发现大部分的数据分析人员现在都会点python。

Python是一种非常友好的语言，虽然对于很多程序员而言无类型让人觉得缺少安全感，但是python的简洁和优雅还是让很多人还是大声喊出了: Life is short, use python!

Spark在设计上就考虑了多语言的支持，除了Scala之外，也提供了java和python的编程接口。

Python On Spark有两种执行方式，一种是pySpark，这是Spark提供的类似于python shell的界面，用户可以通过它交互执行python代码操作数据。另一种方式是用户给定py脚本，提交运行。
<!--more-->
## pySpark架构


我们以pySpark为例分析一下python on spark的实现机制。

我们看一下pyspark的架构:
<img style="mergin:5px;" src="/images/pySpark.png">


(图片来自于
[pyspark wiki](https://cwiki.apache.org/confluence/display/SPARK/PySpark+Internals))


架构上大体把pySpark应用分为Local和Cluster。Local其实也就是Driver端，Cluster就是一系列具体计算进程，一般可以称为Worker或者Executor。

### Local
我们看Local端，当pySpark启动(懒，所以不写如何启动)后，我们可以看到本地其实是成对出现了一个python进程和一个java进程。它们对应图中的python的spark context和java spark context。


<img style="mergin:5px;" src="/images/pySpark-shell.PNG">

这两个python和java进程之间最关键的一点就是通过Py4J来通信。Py4J做好了python对象和java对象之间的转换，这是我们在python上下文中可以自由操作Spark数据对象的基础，毕竟Spark实际还是运行在jvm上。

注意前面架构图中除了socket通信(无论是Py4J还是反向的jvm到python)之外还有通过LocalFS即本地文件系统的通信。这是因为通过Py4J交换大量数据的效率还是比较低下，通过输出为本地文件并进行反序列化就会好很多。


### Cluster

对Cluster端，其实计算程序主要还是运行在jvm之上，和普通的spark程序没有什么区别，但是多了一个阶段是通过Pipe和python进程进行交互。这个思路其实和Hadoop的Streaming/Pipe 是一致的。



## pySpark使用


``` bash
    Welcome to
          ____              __
         / __/__  ___ _____/ /__
        _\ \/ _ \/ _ `/ __/  '_/
       /__ / .__/\_,_/_/ /_/\_\   version 1.6.1
          /_/

    Using Python version 2.7.6 (default, Jun 22 2015 17:58:13)
    SparkContext available as sc, HiveContext available as sqlContext.
```

下面我们举几个例子说明pySpark的使用。

### 操作实例1: Agg操作

我们来实现一个类似于Hive 的UDAF的逻辑: 把一个数据集中的某列String全部拼接起来。


``` python
    from pyspark import Row
    from pyspark.sql.types import *
    #定义两个row，并进而创造出一个DataFrame  
    row = Row(x='hahah')
    row1 = Row(x='ahahaha')
    df = sqlContext.createDataFrame([row, row1])
    #定义一个function把一个list的所有string拼接起来(效率较低，仅为说明用法)
    def concatList(x):
      result = ""
      for i in x:
        result += i
      return result
    #注册function为"concatList"，可以作为udf使用
    sqlContext.registerFunction("concatList", concatList, StringType())
    #对df先调用agg做聚合，把x列用原生函数collect_list构造成为一个list，再转化为
    #新的DataFrame并针对其调用之前注册的concatList就得到了最终的结果。
    df.agg({"x":"collect_list"}).toDF("xlist").selectExpr('concatList(xlist)').collect()
```


### 操作实例2: 表生成操作

我们用pySpark来实现一些类似于Hive UDTF的逻辑。

``` python
    from pyspark.sql.functions import *
    from pyspark import Row
    from pyspark.sql.types import *
    row1 = Row(age = 11, name=['A', 'B', 'C'])
    row2 = Row(age = 21, name=['a', 'b', 'c'])
    df = sqlContext.createDataFrame([row1, row2])
    df.select(df.age, explode(df.name)).collect()
```    


得到结果

``` python
    [Row(age=11, col=u'A'), Row(age=11, col=u'B'), Row(age=11, col=u'C'), Row(age=21, col=u'a'), Row(age=21, col=u'b'), Row(age=21, col=u'c')]
```    




再来个稍微复杂点的例子。

原始数据长这样:

    Login_id    LOGIN_IP                    LOGIN_TIME
    CRonaldo7   192.168.0.1,192.168.0.2     20120101010000,20120102010000

我们想生成:

    Login_id    Login_ip    Login_time
    CRonaldo7   192.168.0.1     20120101010000
    CRonaldo7   192.168.0.2     20120102010000


``` python
    from pyspark.sql.functions import *
    from pyspark import Row
    from pyspark.sql.types import *
    #定义row和dataframe
    row = Row(id='CRonaldo7', ip='192.168.0.1,192.168.0.2', time='20120101010000,20120102010000')
    dfnew = sqlContext.createDataFrame([row])    
    #定义函数，用于把数据重组
    def groupKV(x, y):
        xlist = x.split(',')
        ylist = y.split(',')
        result = []
        for i in range(len(xlist)):
            result.append({xlist[i]:ylist[i]})
        return result

    sqlContext.registerFunction("groupKV", groupKV, ArrayType(MapType(StringType(), StringType())))

    dfmap = dfnew.selectExpr('id', 'groupKV(ip, time)').toDF('id', 'kv')
    dfmap.select(dfmap.id, explode(dfmap.kv)).select('id', explode('col').alias("ip", "time")).collect()
```





### 操作实例3: broadcast Table


熟悉大数据工具Hive的我们都知道，大小表Join的时候可以选择把小表所有结算节点都复制一份变成MapJoin，消除shuffle阶段以提升性能。

Spark SQL 也有这个能力，即所谓BroadcastHashJoin。

通过编程的方式，我们还可以在代码里把整个DataFrame作为广播变量进行广播。


假如我们现在有一个dataframe x长这样:

    [Row(id=u'wangwangA', ip=u'192.168.0.1', time=u'20120101010000'), Row(id=u'wangwangA', ip=u'192.168.0.2', time=u'20120102010000')]


``` python

    from pyspark.sql.functions import *
    from pyspark import Row
    from pyspark.sql.types import *

    ....
    #我们把我们的小小dataframe x 给广播出去，记录为广播变量b  
    b = sc.broadcast(x.collect())


    #定义一个function
    def verify(col):
        for i in b.value:
            if col == i['ip']:
            return 'yes'
        return 'no'


    sqlContext.udf.register("verify", verify, StringType())

    #就可以对一个大的dataframe调用udf，udf内部会去扫描之前广播的x进行处理。
    dfbig.selectExpr("verify(id)").collect()
```



## 实现代码分析

随便摘点实现代码，有兴趣的同学可以继续看看。

### 启动


当你输入pyspark指令时，python环境就启动了，会首先启动${SPARK_HOME}/python/pyspark/shell.py，shell.py会初始化python版本的SparkContext，并在其中打开java gateway，即调用我们很熟悉的spark-submit。


``` python
    def launch_gateway():
        if "PYSPARK_GATEWAY_PORT" in os.environ:
            gateway_port = int(os.environ["PYSPARK_GATEWAY_PORT"])
        else:
            SPARK_HOME = os.environ["SPARK_HOME"]
            # Launch the Py4j gateway using Spark's run command so that we pick up the
            # proper classpath and settings from spark-env.sh
            ....
            command = [os.path.join(SPARK_HOME, script)] + shlex.split(submit_args)
            # Start a socket that will be used by PythonGatewayServer to communicate its port to us
            callback_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            callback_socket.bind(('127.0.0.1', 0))
            callback_socket.listen(1)
            callback_host, callback_port = callback_socket.getsockname()
            env = dict(os.environ)
            env['_PYSPARK_DRIVER_CALLBACK_HOST'] = callback_host
            env['_PYSPARK_DRIVER_CALLBACK_PORT'] = str(callback_port)
```


通过看spark-submit 代码可以找到执行的Java端的主类:


``` java
    if (args.isPython && deployMode == CLIENT) {
        if (args.primaryResource == PYSPARK_SHELL) {
        args.mainClass = "org.apache.spark.api.python.PythonGatewayServer"
        } else {
        // If a python file is provided, add it to the child arguments and list of files to deploy.
        // Usage: PythonAppRunner <main python file> <extra python files> [app arguments]
        args.mainClass = "org.apache.spark.deploy.PythonRunner"
        args.childArgs = ArrayBuffer(args.primaryResource, args.pyFiles) ++ args.childArgs
        if (clusterManager != YARN) {
            // The YARN backend distributes the primary file differently, so don't merge it.
            args.files = mergeFileLists(args.files, args.primaryResource)
        }
        }
```


即Java端入口是PythonGatewayServer.scala。


反过来看python端，launch gateway之后就可以直接“导入”jvm的对象了。


``` python
    # Import the classes used by PySpark
    java_import(gateway.jvm, "org.apache.spark.SparkConf")
    java_import(gateway.jvm, "org.apache.spark.api.java.*")
    java_import(gateway.jvm, "org.apache.spark.api.python.*")
    java_import(gateway.jvm, "org.apache.spark.mllib.api.python.*")
    # TODO(davies): move into sql
    java_import(gateway.jvm, "org.apache.spark.sql.*")
    java_import(gateway.jvm, "org.apache.spark.sql.hive.*")
    java_import(gateway.jvm, "scala.Tuple2")

```



### SQL UDF

目前python 方式的SQL UDF只支持普通UDF，不支持UDAF和UDTF。我们看看在python上下文里是如何注册一个udf的吧。


我们看sql\context.py里，registerFunction:

``` python
    udf = UserDefinedFunction(f, returnType, name)
    self._ssql_ctx.udf().registerPython(name, udf._judf)
```


我们发现其实还是通过java的对象在操作。


我们先看看这个python的UserDefinedFunction对象是什么。


``` python
def _create_judf(self, name):
    from pyspark.sql import SQLContext
    f, returnType = self.func, self.returnType  # put them in closure `func`
    #这里构造了闭包func。其实是迭代执行f的过程。注意要把pySpark类型转为#python内部类型
    
   func = lambda _, it: map(lambda x: returnType.toInternal(f(*x)), it)           
   # 序列化方法 
   ser = AutoBatchedSerializer(PickleSerializer())
   command = (func, None, ser, ser)
   sc = SparkContext.getOrCreate()
   #一系列参数准备。pickled_command是序列化之后的command
   pickled_command, broadcast_vars, env, includes = _prepare_for_python_RDD(sc, command, self)
   ctx = SQLContext.getOrCreate(sc)
   #jdt是解析后的Data Type
   jdt = ctx._ssql_ctx.parseDataType(self.returnType.json())
   ....
   #注意最终其实产生的还是个java对象。                
   judf = sc._jvm.UserDefinedPythonFunction(name, bytearray(pickled_command), env, includes,sc.pythonExec, sc.pythonVer, broadcast_vars,sc._javaAccumulator, jdt)
   return judf
```



我们看UDFRegistration中的registerPython:


    functionRegistry.registerFunction(name, udf.builder)




实际上这个java版本的python udf是通过builtin的FunctionRegistry即SimpleFunctionRegistry进行注册的。


怎么运行的可以看这段注释(python.scala)，通俗易懂。


    /**
     - Uses PythonRDD to evaluate a [[PythonUDF]], one partition of tuples at a time.
     *
     - Python evaluation works by sending the necessary (projected) input data via a socket to an
     - external Python process, and combine the result from the Python process with the original row.
     *
     - For each row we send to Python, we also put it in a queue. For each output row from Python,
     - we drain the queue to find the original input row. Note that if the Python process is way too
     - slow, this could lead to the queue growing unbounded and eventually run out of memory.
     */



注意看，BatchPythonEvaluation->doExecute, 这是Cluster端的事情了。


``` scala
    // Output iterator for results from Python.
        val outputIterator = new PythonRunner(
            udf.command,
            udf.envVars,
            udf.pythonIncludes,
            udf.pythonExec,
            udf.pythonVer,
            udf.broadcastVars,
            udf.accumulator,
            bufferSize,
            reuseWorker
        ).compute(inputIterator, context.partitionId(), context)
```

这里就是之前提到的pipe的概念，我们再看worker.py，一切就对得上了。




    func, profiler, deserializer, serializer = command



最终把原始python 的function给“重放”出来……


``` python
    def process():
                iterator = deserializer.load_stream(infile)
                serializer.dump_stream(func(split_index, iterator), outfile)

```





## 总结


Spark 的开发者借助Py4J这样的工具和Hadoop Streaming/Pipe的思想，以较小的代价(一头一尾)实现了python语言的支持，通过python的编程接口，降低了数据分析的学习成本，是很不错的思路。

