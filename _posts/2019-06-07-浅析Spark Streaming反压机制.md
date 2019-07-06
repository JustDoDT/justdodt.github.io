---

layout:     post
title:      "浅析Spark Streaming反压机制"
date:       2019-06-07 02:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - spark
---



### 反压机制原理

Spark Streaming中的反压机制是Spark 1.5.0推出的新特性，可以根据处理效率动态调整摄入速率。

当批处理时间（Batch Processing Time）大于批次间隔(Batch Interval，即BatchDuration)时，说明处理数据的速度小于数据摄入的速度，持续时间过长或数据源头数据暴增，容易造成数据在内存的堆积，最终导致Executor OOM或任务崩溃。

在这种情况下，若是基于Kafka Receiver的数据源，可以通过设置spark.streaming.receiver.maxRate来控制最大输入速率；若是基于Direct的数据源(如Kafka Direct Stream)，则可以通过设置spark.streaming.maxRatePerPartition来控制最大输入速率。当然，在事先经过压测，且流量高峰不会超过预期的情况下，设置这些参数一般没有什么问题。但是最大值，不代表是最优值，最好还能根据每个批次处理情况动态预估下个批次的最优速率。在Spark1.5.0 以上，就可通过背压机制来实现。开启反压机制，即设置spark.streaming.backpressure.enabled为true，Spark Streaming会自动根据处理能力来调整输入速率，从而在流量高峰时仍能保证最大的吞吐和性能。

**部分代码**

~~~
override def onBatchCompleted(batchCompleted: StreamingListenerBatchCompleted) {
    val elements = batchCompleted.batchInfo.streamIdToInputInfo

    for {
      // 处理结束时间
      processingEnd <- batchCompleted.batchInfo.processingEndTime
      // 处理时间,即`processingEndTime` - `processingStartTime`
      workDelay <- batchCompleted.batchInfo.processingDelay
      // 在调度队列中的等待时间,即`processingStartTime` - `submissionTime`
      waitDelay <- batchCompleted.batchInfo.schedulingDelay
      // 当前批次处理的记录数
      elems <- elements.get(streamUID).map(_.numRecords)
    } computeAndPublish(processingEnd, elems, workDelay, waitDelay)

~~~



可以看到，接着又调用的是`computeAndPublish`方法，如下：

~~~
private def computeAndPublish(time: Long, elems: Long, workDelay: Long, waitDelay: Long): Unit =
    Future[Unit] {
      // 根据处理时间、调度时间、当前Batch记录数，预估新速率
      val newRate = rateEstimator.compute(time, elems, workDelay, waitDelay)
      newRate.foreach { s =>
      // 设置新速率
        rateLimit.set(s.toLong)
      // 发布新速率
        publish(getLatestRate())
      }
    }

~~~



更深一层，具体调用的是`rateEstimator.compute`方法来预估新速率，如下：

~~~
def compute(
      time: Long,
      elements: Long,
      processingDelay: Long,
      schedulingDelay: Long): Option[Double]
~~~



该方法是接口RateEstimator中的方法，会计算出新的批次每秒摄入的记录数。PIDRateEstimator,即PID速率估算器，是RateEstimator的唯一实现，具体估算逻辑可看PIDRateEstimator.compute方法，逻辑很复杂，用到了微积分相关的知识，总之，一句话，即根据当前Bath的结果和期望的差值来估算新的输入速率。

### 反压机制相关的参数

1.`spark.streaming.backpressure.enabled`

默认值是false,是否开启反压机制。

2.`spark.streaming.backpressure.initialRate`

默认值无，初始最大接收速率。只适用于Receiver Stream,不适用于Direct Stream。类型为整数，默认直接读取所有，在1开启的情况下，限制第一次批处理应该消费数据，因为程序冷启动有大量的挤压，防止第一次全部读取，造成系统阻塞。

3.`spark.streaming.kafka.maxRatePerPartition`

类型为整数,默认直接读取所有,限制每秒每个消费线程读取每个kafka分区最大的数据量

4.`spark.streaming.stopGracefullyOnShutdown`

优雅关闭，确保在kill任务时，能够处理完最后一批数据，再关闭程序，不会发生强制kill导致数据处理中断，没有处理完的数据丢失。

**注意：**


`只有 3 激活的时候，每次消费的最大数据量，就是设置的数据量，如果不足这个数，就有多少读多少，如果超过这个数字，就读取这个数字的设置的值。`

`只有 1 + 3 激活的时候，每次消费读取我的数量最大会等于3设置的值，最小是spark根据系统负载自动推断的值，消费的数据量会在这两个范围之内变化根据系统的情况，但是第一次启动会有多少读多少数据。此后安装 1+3 设置规则运行。`

`1 + 2 + 3 同时激活的时候，跟上一个消费者情况基本一样，但第一次消费会得到限制，因为我们设置第一次消费的频率了。`




5.`spark.streaming.backpressure.rateEstimator`

默认值pid,速率控制器，Spark默认只支持此控制器，可自定义。

6.`spark.streaming.backpressure.pid.proportional`

默认值1.0，只能为非负值。当前速率于最后一批速率之间的差值对总控制信号贡献的权重。用默认值即可。

7.`spark.streaming.backpressure.pid.integral`

默认值0.2，只能为非负值。比例误差累积对总控制信号贡献的权重。用默认值即可。

8.`spark.streaming.backpressure.pid.derived`

默认值0.0，只能为非负值。比例误差变化对总控制信号贡献的权重。用默认值即可。

9.`spark.streaming.backpressure.pid.minRate`

默认值100，只能为正数，最小速率。

### 反压机制的使用

~~~
//启用反压机制
conf.set("spark.streaming.backpressure.enabled","true")
//最小摄入条数控制
conf.set("spark.streaming.backpressure.pid.minRate","1")
//最大摄入条数控制
conf.set("spark.streaming.kafka.maxRatePerPartition","12")
//初始最大接收速率控制
conf.set("spark.streaming.backpressure.initialRate","10")    

~~~



要保证反压机制真正起作用前Spark应用程序不会崩溃，需要控制每个批次最大摄入速率。以Direct Stream为例，如Kafka Direct Stream，则可以通过

spark.streaming.kafka.maxRatePerPartition参数来控制。此参数代表了每秒每个分区最大摄入的数据条数。假设BatchDuration为10秒，

spark,streaming.kafka.maxRatePerPartition为12条，kafka topic分区数为3个，则一个批次(Batch) 最大读取的数据条数为`360条(3 * 12 * 10=360)`。同时，

需要注意的是，该参数也代表了整个应用生命周期的最大速率即使是反压调整的最大值也不会超过这个参数。



















