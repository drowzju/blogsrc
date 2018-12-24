---
title: Spark 从JSON加载数据
author: 梅溪
comments: true
layout: post
slug: 
tags : [大数据,Spark]
date: 2016-06-06 16:01:46
---

# spark load json

JSON(JavaScript Object Notation) 是一种轻量级的数据交换格式，在互联网公司尤为常见。但是JSON并不能视为一种严格的结构化数据，要想导入其他系统如Hive需要一些额外的解析转换的编码工作。好在我们现在有Spark，它的DataFrame接口让我们可以更方便地处理JSON格式的数据。

本文将以scala代码展示Spark 1.6.1上如何加载并处理JSON格式的数据。
<!--more-->
## JSON数据加载

启动Spark shell后，下面两行代码，就可以从JSON文件加载数据存为一个Dataframe df。/tmp/new.json是一个HDFS的路径。


``` scala    
    val sqlContext = new org.apache.spark.sql.SQLContext(sc)
    val df = sqlContext.read.json("/tmp/new.json")
```



第一行是生成一个Spark的SQLContext。在Spark中，当你需要处理带schema的数据，你都需要一个SQLContext(HiveContext是一种兼容了Hive的SQLContext)。
第二行则是使用当前SQLContext的DataframeReader从指定位置加载JSON文件得到一个Dataframe对象df。

后面我们拿着df就可以做各种操作啦。

加载json还可以带各种选项，可以参见Spark的API文档：
    
``` scala    
    primitivesAsString (default false): infers all primitive values as a string type
    allowComments (default false): ignores Java/C++ style comment in JSON records
    allowUnquotedFieldNames (default false): allows unquoted JSON field names
    allowSingleQuotes (default true): allows single quotes in addition to double quotes
    allowNumericLeadingZeros (default false): allows leading zeros in numbers (e.g. 00012) 
```



## 数据内容和Schema展示


打印schema。太长我略去了部分。这里我用的JSON文件导致展示出来都是String类型，为什么这样我们稍后再说。


``` scala
    scala> df.printSchema
    root
     |-- CDBH: string (nullable = true)
     |-- CDLX: string (nullable = true)
     |-- CLLX: string (nullable = true)
     |-- CLPP: string (nullable = true)
     |-- CLSD: string (nullable = true)
     |-- CLWX: string (nullable = true)
     .....
     |-- TX8: string (nullable = true)
     |-- TXSL: string (nullable = true)
     |-- WFZT: string (nullable = true)
     |-- XSZT: string (nullable = true)
     |-- XXBH: string (nullable = true)
     |-- _corrupt_record: string (nullable = true)
```



注意这里有个"_corrupt_record"，这是解析的时候出现了无法解析的列，一般是JSON文件格式的问题。
这种方式加载JSON文件，要求文件内容必须是连续的若干行，即连续的多个"{...}"，以逗号隔开。
最好采集数据时能够按照要求组织好（并不难），如果不能符合要求，也可以采集之后，加载之前对数据做点加工。

比如我就是写了几行python脚本，把末尾的多余字符裁掉。

``` python    
    f = open("new.json", "r+")
    f.seek(-13, 2)
    f.truncate(f.tell())
    f.close()
```

当然，另一种做法是先不管这些，后面有了df之后再用select或别的方式忽略掉错误的列。




## 指定Schema


Spark加载JSON时会根据输入的数据来推导数据类型。
这种推导一般能够正常工作，不过也许我们需要显式地要求一个schema。这时候我们可以这样： 

``` scala    
    import org.apache.spark.sql.types.{StructType,StructField,FloatType}
    //要求CLSD，CLXS这两列是Float类型，声明schema
    val schemaString = "CLSD CLXS"
    val schema = StructType(schemaString.split(" ").map(fieldName => StructField(fieldName, FloatType, true)))
    val df = sqlContext.read.schema(schema).json("/tmp/new.json")
```


我实际操作时还遇到问题，是JSON数据中全是"foo":"2"的形式(冒号后的value全部用引号引起来)，这样一来，按照JSON的标准，value会全部推导成String类型，导致我指定schema也无效，甚至会解析出错。这个问题留在下一节解决。


## 处理数据

加载数据只是第一步，真正强大的处理能力才是我们更看重的。


Dataframe最妙的就是它是一个带有schema针对数据record的抽象，可以自由地使用SQL操作：

比如针对"HPYS"这一列做转换：


    df.select(expr("conv(HPYS, 10, 10) as HPYS")).show
    
    +----+
    |HPYS|
    +----+
    |   2|
    ...
    |   2|
    |   2|
    |   4|
    +----+
    only showing top 20 rows
    



我们可以对Dataframe做各种各样的转换，甚至是修改列的类型，解决前一节所遇到的JSON文件不标准导致的所有列都推导成String的问题。

把HPYS 这一列转成int

    
    scala> df.withColumn("HPYS", df("HPYS").cast("int")).printSchema
    root
     |-- CDBH: string (nullable = true)
    ....
     |-- CWHPHM: string (nullable = true)
     |-- CWHPYS: string (nullable = true)
     |-- FXBH: string (nullable = true)
     |-- HPHM: string (nullable = true)
     |-- HPYS: integer (nullable = true)
    





## 数据写回

处理好的数据写回去也很简单。
SparkSQL会以Parquet作为默认的输出格式。如果你希望指定格式，可以调用format方法：

``` scala
df.withColumn("HPYS", df("HPYS").cast("int")).write.save("/tmp/int.json")
df.withColumn("HPYS", df("HPYS").cast("int")).write.format("json").save("/tmp/int.json")
```






## 小结

本文总结了spark加载JSON数据并进行处理的方法，希望对你有所帮助。如有问题欢迎指出讨论。

