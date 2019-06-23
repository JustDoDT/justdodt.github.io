---
layout:     post
title:      "kafka+Spark Streaming管理offset的方法"
date:       2019-05-28 02:41:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Kafka

---



### 概述

kafka配合Spark Streaming是大数据领域常见的黄金搭档，主要用于数据实时处理。为了应对可能出现的引起Streaming程序崩溃的异常情况，我们一般都需要手动管理好Kafka的offset，而不是让它自动提交，即需要将`enable.auto.commit`设为false。只有管理好offset，才能使整个流式系统最大限度地接近exactly once语义。

**kafka提供了三种语义的传递**

- 至少一次

- 至多一次

- 精确一次

  

 - 首先在producer端保证“至少一次”和“至多一次”语义是非常简单的，至少一次只需要同步确认即可（确认方式分为只需要leader确认以及所有副本都确认，第二种更加具有容错性）；至多一次最简单，只需要异步不断的发送即可，效率也比较高。目前在producer端还不能保证精确一次，在未来有可能实现，实现方式如下：在同步确认的基础上为每一条消息加一个主键，如果发现主键曾经接受过，则丢弃。

 - 在 consumer 端，大家都知道可以控制 offset，所以可以控制消费，其实 offset 只有在重启的时候才会用到。在机器正常运行时我们用的是 position，我们实时消费的位置也是 position 而不是 offset。我们可以得到每一条消息的 position。如果我们在处理消息之前就将当前消息的 position 保存到 zk 上即 offset，这就是`至多一次`消费，因为我们可能保存成功后，消息还没有消费机器就挂了，当机器再打开时此消息就丢失了；或者我们可以先消费消息然后保存 position 到 zk 上即 offset，此时我们就是`至少一次`，因为我们可能在消费完消息后offset 没有保存成功。而`精确一次`的做法就是让 position的保存和消息的消费成为原子性操作，比如将消息和 position 同时保存到 hdfs 上 ，此时保存的 position 就称为 offset，当机器重启后，从 hdfs重新读入offset，这就是精确一次。



- consumer可以先读取消息，然后将offset写入日志文件中，然后再处理消息。这存在一种可能就是在存储offset后还没处理消息就crash了，新的consumer继续从这个offset处理，那么就会有些消息永远不会被处理，这就是上面说的“最多一次”。
- consumer可以先读取消息，处理消息，最后记录offset，当然如果在记录offset之前就crash了，新的consumer会重复的消费一些消息，这就是上面说的“最少一次”。
- “精确一次”可以通过将提交分为两个阶段来解决：保存了offset后提交一次，消息处理成功之后再提交一次。但是还有个更简单的做法：将消息的offset和消息被处理后的结果保存在一起。比如用Hadoop ETL处理消息时，将处理后的结果和offset同时保存在HDFS中，这样就能保证消息和offser同时被处理了。

### 管理offset的流程

下面这张图能够简要的说明管理offset的大致流程。


![SparkStreaming](/img/Spark/SparkStreaming/SparkStreaming1.png)



- 在Kafka DirectStream初始化时，取得当前所有partition的存量offset，以让DirectStream能够从正确的位置开始读取数据。

- 读取消息数据，处理并存储结果。

- 提交offset，并将其持久化在可靠的外部存储中。
   图中的“process and store results”及“commit offsets”两项，都可以施加更强的限制，比如存储结果时保证幂等性，或者提交offset时采用原子操作。
   图中提出了4种offset存储的选项，分别是HBase、Kafka自身、HDFS和ZooKeeper。综合考虑实现的难易度和效率，我们目前采用过的是Kafka自身与ZooKeeper两种方案。



### kafka自身

在Kafka 0.10+版本中，offset的默认存储由ZooKeeper移动到了一个自带的topic中，名为__consumer_offsets。Spark Streaming也专门提供了commitAsync() API用于提交offset。使用方法如下。

~~~
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  // 确保结果都已经正确且幂等地输出了
  stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
~~~



上面是Spark Streaming官方文档中给出的写法。但在实际上我们总会对DStream进行一些运算，这时我们可以借助DStream的transform()算子。

~~~
var offsetRanges: Array[OffsetRange] = Array.empty[OffsetRange]

 stream.transform(rdd => {
     // 利用transform取得OffsetRanges
     offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
     rdd
 }).mapPartitions(records => {
     var result = new ListBuffer[...]()
     // 处理流程
     result.toList.iterator
 }).foreachRDD(rdd => {
     if (!rdd.isEmpty()) {
         // 数据入库
         session.createDataFrame...
     }
     // 提交offset
     stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
 })
~~~



特别需要注意，在转换过程中不能破坏RDD分区与Kafka分区之间的映射关系。亦即像map()/mapPartitions()这样的算子是安全的，而会引起shuffle或者repartition的算子，如reduceByKey()/join()/coalesce()等等都是不安全的。
 另外需要注意的是，`HasOffsetRanges`是`KafkaRDD`的一个trait，而`CanCommitOffsets`是`DirectKafkaInputDStream`的一个trait。从spark-streaming-kafka包的源码中，可以看得一清二楚。



~~~
private[spark] class KafkaRDD[K, V](
    sc: SparkContext,
    val kafkaParams: ju.Map[String, Object],
    val offsetRanges: Array[OffsetRange],
    val preferredHosts: ju.Map[TopicPartition, String],
    useConsumerCache: Boolean
) extends RDD[ConsumerRecord[K, V]](sc, Nil) with Logging with HasOffsetRanges

private[spark] class DirectKafkaInputDStream[K, V](
    _ssc: StreamingContext,
    locationStrategy: LocationStrategy,
    consumerStrategy: ConsumerStrategy[K, V],
    ppc: PerPartitionConfig
  ) extends InputDStream[ConsumerRecord[K, V]](_ssc) with Logging with CanCommitOffsets {
~~~



这就意味着不能对stream对象做transformation操作之后的结果进行强制转换（会直接报ClassCastException），因为RDD与DStream的类型都改变了。只有RDD或DStream的包含类型为ConsumerRecord才行。



### ZooKeeper

虽然Kafka将offset从ZooKeeper中移走是考虑到可能的性能问题，但ZooKeeper内部是采用树形node结构存储的，这使得它天生适合存储像offset这样细碎的结构化数据。并且我们的分区数不是很多，batch间隔也相对长（20秒），因此并没有什么瓶颈。
 Kafka中还保留了一个已经标记为过时的类`ZKGroupTopicDirs`，其中预先指定了Kafka相关数据的存储路径，借助它，我们可以方便地用ZooKeeper来管理offset。为了方便调用，将存取offset的逻辑封装成一个类如下。

~~~
class ZkKafkaOffsetManager(zkUrl: String) {
    private val logger = LoggerFactory.getLogger(classOf[ZkKafkaOffsetManager])

    private val zkClientAndConn = ZkUtils.createZkClientAndConnection(zkUrl, 30000, 30000);
    private val zkUtils = new ZkUtils(zkClientAndConn._1, zkClientAndConn._2, false)

    def readOffsets(topics: Seq[String], groupId: String): Map[TopicPartition, Long] = {
        val offsets = mutable.HashMap.empty[TopicPartition, Long]
        val partitionsForTopics = zkUtils.getPartitionsForTopics(topics)

        // /consumers/<groupId>/offsets/<topic>/<partition>
        partitionsForTopics.foreach(partitions => {
            val topic = partitions._1
            val groupTopicDirs = new ZKGroupTopicDirs(groupId, topic)

            partitions._2.foreach(partition => {
                val path = groupTopicDirs.consumerOffsetDir + "/" + partition
                try {
                    val data = zkUtils.readData(path)
                    if (data != null) {
                        offsets.put(new TopicPartition(topic, partition), data._1.toLong)
                        logger.info(
                            "Read offset - topic={}, partition={}, offset={}, path={}",
                            Seq[AnyRef](topic, partition.toString, data._1, path)
                        )
                    }
                } catch {
                    case ex: Exception =>
                        offsets.put(new TopicPartition(topic, partition), 0L)
                        logger.info(
                            "Read offset - not exist: {}, topic={}, partition={}, path={}",
                            Seq[AnyRef](ex.getMessage, topic, partition.toString, path)
                        )
                }
            })
        })

        offsets.toMap
    }

    def saveOffsets(offsetRanges: Seq[OffsetRange], groupId: String): Unit = {
        offsetRanges.foreach(range => {
            val groupTopicDirs = new ZKGroupTopicDirs(groupId, range.topic)
            val path = groupTopicDirs.consumerOffsetDir + "/" + range.partition
            zkUtils.updatePersistentPath(path, range.untilOffset.toString)
            logger.info(
                "Save offset - topic={}, partition={}, offset={}, path={}",
                Seq[AnyRef](range.topic, range.partition.toString, range.untilOffset.toString, path)
            )
        })
    }
}
~~~



这样，offset就会被存储在ZK的/consumers/[groupId]/offsets/[topic]/[partition]路径下。当初始化DirectStream时，调用readOffsets()方法获得offset。当数据处理完成后，调用saveOffsets()方法来更新ZK中的值。



### Offset存储到外部存储系统中

#### 存到MySQL中

参考博客:    [SparkStreaming数据零丢失使用mysql存储kafka的offset](<https://justdodt.github.io/2019/05/27/SparkStreaming%E6%95%B0%E6%8D%AE%E9%9B%B6%E4%B8%A2%E5%A4%B1%E4%BD%BF%E7%94%A8mysql%E5%AD%98%E5%82%A8kafka%E7%9A%84offset/>)

#### 存储到HBase中

HBase可以作为一个可靠的外部数据库来持久化offsets。通过将offsets存储在外部系统中，Spark Streaming应用功能能够重读或者回放任何仍然存储在Kafka中的数据。

根据HBase的设计模式，允许应用能够以rowkey和column的结构将多个Spark Streaming应用和多个Kafka topic存放在一张表格中。在这个例子中，表格以topic名称、消费者group id和Spark Streaming 的`batchTime.milliSeconds`作为rowkey以做唯一标识。尽管`batchTime.milliSeconds`不是必须的，但是它能够更好地展示历史的每批次的offsets。表格将存储30天的累积数据，如果超出30天则会被移除。



### 为什么不用checkpoint

Spark Streaming的checkpoint机制无疑是用起来最简单的，checkpoint数据存储在HDFS中，如果Streaming应用挂掉，可以快速恢复。
 但是，如果Streaming程序的代码改变了，重新打包执行就会出现反序列化异常的问题。这是因为checkpoint首次持久化时会将整个jar包序列化，以便重启时恢复。重新打包之后，新旧代码逻辑不同，就会报错或者仍然执行旧版代码。
 要解决这个问题，只能将HDFS上的checkpoint文件删掉，但这样也会同时删掉Kafka的offset信息，就毫无意义了。



### 总结

- Kafka版本[[0.10.1.1](http://kafka.apache.org/downloads)]，已默认将消费的 offset 迁入到了 Kafka 一个名为 __consumer_offsets 的Topic中。其实，早在 0.8.2.2 版本，已支持存入消费的 offset 到Topic中，只是那时候默认是将消费的 offset 存放在 Zookeeper 集群中。那现在，官方默认将消费的offset存储在 Kafka 的Topic中，同时，也保留了存储在 Zookeeper 的接口，通过 offsets.storage 属性来进行设置。

- 其实，官方这样推荐，也是有其道理的。之前版本，Kafka其实存在一个比较大的隐患，就是利用 Zookeeper 来存储记录每个消费者/组的消费进度。虽然，在使用过程当中，JVM帮助我们完成了一些优化，但是消费者需要频繁的去与 Zookeeper 进行交互，而利用ZKClient的API操作Zookeeper频繁的Write其本身就是一个比较低效的Action，对于后期水平扩展也是一个比较头疼的问题。如果期间 Zookeeper 集群发生变化，那 Kafka 集群的吞吐量也跟着受影响。

- 在此之后，官方其实很早就提出了迁移到 Kafka 的概念，只是，之前是一直默认存储在 Zookeeper集群中，需要手动的设置，如果，对 Kafka 的使用不是很熟悉的话，一般我们就接受了默认的存储（即：存在 ZK 中）。在新版 Kafka 以及之后的版本，Kafka 消费的offset都会默认存放在 Kafka 集群中的一个叫 __consumer_offsets 的topic中。

- 当然，其实她实现的原理也让我们很熟悉，利用 Kafka 自身的 Topic，以消费的Group，Topic，以及Partition做为组合 Key。所有的消费offset都提交写入到上述的Topic中。因为这部分消息是非常重要，以至于是不能容忍丢数据的，所以消息的 acking 级别设置为了 -1，生产者等到所有的 ISR 都收到消息后才会得到 ack（数据安全性极好，当然，其速度会有所影响）。所以 Kafka 又在内存中维护了一个关于 Group，Topic 和 Partition 的三元组来维护最新的 offset 信息，消费者获取最新的offset的时候会直接从内存中获取。  



### 参考文章

- [Spark Streaming 管理 Kafka Offsets 的方式探讨](<https://www.jianshu.com/p/ef3f15cf400d>)

- [Offset Management For Apache Kafka With Apache Spark Streaming](<https://blog.cloudera.com/blog/2017/06/offset-management-for-apache-kafka-with-apache-spark-streaming/>)





























