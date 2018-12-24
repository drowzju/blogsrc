
# SparkSQL Adaptive Execution

## Why
Spark 总是需要为job提前生成执行的DAG。即便有CBO，也很难提前知道数据和UDF的情况。

Spark比数据库类型的workloads更需要adaptive query execution，因为Spark几乎总是在处理新数据，没有机会去index，而且UDF更具多样性。


## Goal
目标是:

- DAGScheduler 提供内部API，可以独立提交DAG stage并收集它们的执行结果
- 具备改变下游stage 交互行为的能力。比如改变reduce 任务的数目，或广播一些数据而不是shuffle
- 修改Spark SQL的Planner，可以设置agg的reduce任务数目，选择join策略


## Architecture

基本思路是在决定reduce端策略前先执行map端。

在map端会先产生足够大的分区数目(比如1000)，然后看这些分区输出的大小，决定
使用多少reduce任务，每个reduce任务要取那些分区的数据，或者是否要广播。要实现这些，reduce任务要有能力去获取多个map 输出分区的数据。


举例，比如A，B两表join，各产生1000个输出分区，可以有三种策略:

- shuffle join, 每个reduce 任务从A，B的输出各选取一些输出分区进行计算。driver决定reduce 任务的数目，以及哪些输出分区到特定reduce 任务。
- 把A的输出广播
- 广播A的一些分区


实现分解:

- DAG Scheduler: 允许单独提交map的stage，并收集统计信息
- Shuffle:  允许reduce 任务获取多个map输出分区
- Spark SQL: 实现具体的adaptive query planning
    

### DAG Scheduler

增加DAGScheduler.submitMapStage()方法，接受代表给定map stage的ShuffleDependency，执行任务并返回一个MapOutputStatistics对象。

      def submitMapStage[K, V, C](
          dependency: ShuffleDependency[K, V, C],
          callback: MapOutputStatistics => Unit,
          callSite: CallSite,
          properties: Properties): JobWaiter[MapOutputStatistics] = {


当map stage结束(或者再早一些)，下游stage就可以以相同的ShuffleDependency 提交，并自动纳入DAGScheduler的dependency tracking。



### Shuffle Layer

除了请求多个block之外，还有两个优化会比较有用:

- 允许shuffle读者要求一些block在接收节点做缓存(方便broadcast复用) 
- 优化block manager的shuffle block server，允许一次disk read获取多个分区id




### Spark SQL

增加一种Exchange operator，我们不妨称之为AdaptiveExchange。为它的子operators提交ShuffleMapStages并基于统计信息创建一个ShuffledRDD。

目前只是针对DataFrame的action应用adaptive query execution。



#### 运行时决策

- 设置Reducer数目。设定一个相对大的分区数目和一个合理的单Reducer处理数据大小
- 决定join策略。shuffle join->broadcast join
- 处理数据倾斜。不同分区数据量差异很大，小的则broadcast，大的也不要整个儿shuffle。



#### 修改

##### 查询编译

在查询编译时，当我们发现一个physical operator定义了requiredChildDistribution(表示此operator对它的输入row如何组织有特殊需求)，就添加一个AdaptiveExchange operator.

This logic is different from our
current physical planning logic, which uses EnsureRequirements to add Exchange operators
and tries to avoid add unnecessary Exchange operators.

At
runtime, an AdaptiveExchange operator will determine if to shuffle data based on the
outputPartitioning of its child and its requiredChildDistribution field.



一个例子。

join->agg

In the query plan for adaptive query execution, the AdaptiveExchange operator appearing
before the Join operator has two children and it can plan how to shuffle the data by considering
stats from both table A and table B. The AdaptiveExchange operator appearing before the
Aggregate operator will check if its input rows already satisfy the required distribution. If so, it is
a NoOp operator.

##### Runtime Execution

AdaptiveExchange的 doExecute方法调用后，首先会获得子operator的RDD，并提交ShuffleMapStages；
之后会阻塞住直到这些stages结束。

提交的stage完成之后，AdaptiveExchange搜集统计信息并调用它的planning策略，决定如何创建ShuffledRDD以获取map输出(是reduce阶段的首个RDD)。

最后，outputPartitioning 被更新(比如设置实际使用的分区数目)，并返回创建的ShuffledRDD.

##### Tasks


- 实现AdaptiveExchange operator，并挂到query planner。有feature flag。
- 对不同的operator，在AdaptiveExchange operator中添加不同的planning策略。
- 扩展实现新的可以支持AdaptiveExchange operator的operators
- 设置合理的一些标量(比如每个reduce处理的数据数量)





##  Limitations and Extensions



当前最主要限制是我们需要大量的map输出分区。将来也许可以实现成一个排序后的大文件然后带着分区offset。



我们可能的扩展

- 扩展到streaming
- 应用到core
- 利用更多统计信息，而不至于size
- 更多SQL join类型？



## 参考

https://issues.apache.org/jira/browse/SPARK-9850



https://github.com/Intel-bigdata/spark-adaptive

    