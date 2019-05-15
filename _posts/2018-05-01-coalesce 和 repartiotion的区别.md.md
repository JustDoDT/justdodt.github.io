---
layout:     post
title:      "coalesce和repartiotion的区别"
date:       2018-05-01 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - spark
---


### 1. coalesce 和 repartiotion的区别

#### 1.1 coalesce在源码中的介绍


        /**
           * Return a new RDD that is reduced into `numPartitions` partitions.
           *
           * This results in a narrow dependency, e.g. if you go from 1000 partitions
           * to 100 partitions, there will not be a shuffle, instead each of the 100
           * new partitions will claim 10 of the current partitions. If a larger number
           * of partitions is requested, it will stay at the current number of partitions.
           *
           * However, if you're doing a drastic coalesce, e.g. to numPartitions = 1,
           * this may result in your computation taking place on fewer nodes than
           * you like (e.g. one node in the case of numPartitions = 1). To avoid this,
           * you can pass shuffle = true. This will add a shuffle step, but means the
           * current upstream partitions will be executed in parallel (per whatever
           * the current partitioning is).
           *
           * @note With shuffle = true, you can actually coalesce to a larger number
           * of partitions. This is useful if you have a small number of partitions,
           * say 100, potentially with a few partitions being abnormally large. Calling
           * coalesce(1000, shuffle = true) will result in 1000 partitions with the
           * data distributed using a hash partitioner. The optional partition coalescer
           * passed in must be serializable.
           */
          def coalesce(numPartitions: Int, shuffle: Boolean = false,
                partitionCoalescer: Option[PartitionCoalescer] = Option.empty)
               (implicit ord: Ordering[T] = null)
              : RDD[T] = withScope {
            require(numPartitions > 0, s"Number of partitions ($numPartitions) must be positive.")




#### 1.2 repartition在源码中的介绍


        /**
           * Return a new RDD that has exactly numPartitions partitions.
           *
           * Can increase or decrease the level of parallelism in this RDD. Internally, this uses
           * a shuffle to redistribute data.
           *
           * If you are decreasing the number of partitions in this RDD, consider using `coalesce`,
           * which can avoid performing a shuffle.
           */
          def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
            coalesce(numPartitions, shuffle = true)
          }
    



`从源码中可以看出，repartition底层是调用的coalesce；repatition是一定要经过shuffle的，coalesce在分区数目减少的情况下不经过shuffle，在分区数目增加的情况下也是会产生shuffle的。`



### 2. 测试

~~~scala
scala> val data = sc.parallelize(List(1,2,3,4))
data: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[6] at parallelize at <console>:24

scala> data.partitions.length
res6: Int = 4

scala> val data1 = data.coalesce(1)
data1: org.apache.spark.rdd.RDD[Int] = CoalescedRDD[7] at coalesce at <console>:25

scala> data1.partitions.length
res7: Int = 1

scala> val data2 = data.coalesce(5)
data2: org.apache.spark.rdd.RDD[Int] = CoalescedRDD[8] at coalesce at <console>:25

scala> data2.partitions.length
res8: Int = 4

scala> val data3 = data.coalesce(5,true)
data3: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[12] at coalesce at <console>:25

scala> data3.partitions.length
res9: Int = 5

scala> data3.collect
res10: Array[Int] = Array(1, 3, 4, 2)

scala> data1.collect
res11: Array[Int] = Array(1, 2, 3, 4)
~~~

**当data3（即分区数目变大）触发job时的DAG图形**

![浅谈RDD](/img/Spark/coalesce1.png)  


**当data1（即分区数目变小）触发job时候的DAG图形**

![浅谈RDD](/img/Spark/coalesce2.png)  


**repartition测试，当分区数目变大**

~~~scala
scala> data.partitions.length
res13: Int = 4

scala> val data4 = data.repartition(6)
data4: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[16] at repartition at <console>:25

scala> data4.collect
res14: Array[Int] = Array(1, 4, 3, 2)

~~~



![浅谈RDD](/img/Spark/coalesce3.png)  


**repartition测试，当分区数目减少的时候**

~~~scala
scala> val data5 = data.repartition(2)
data5: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[20] at repartition at <console>:25

scala> data5.partitions.length
res16: Int = 2

scala> data5.collect
res15: Array[Int] = Array(1, 3, 4, 2)
~~~





![浅谈RDD](/img/Spark/coalesce4.png)  


**不使用coalesce和repartition时候**

~~~scala
scala> import scala.collection.mutable.ListBuffer
import scala.collection.mutable.ListBuffer

scala> val students = sc.parallelize(List("A","B","C","D","E","F"),3)
students: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[23] at parallelize at <console>:25

scala> students.mapPartitionsWithIndex((index,paritions) =>{
     |   val stus = new ListBuffer[String]
     |   while (paritions.hasNext){
     |     stus += (paritions.next() + "\t" + (index + 1) + "组" )
     |   }
     |   stus.iterator
     | }).foreach(println)
E       3组
F       3组
A       1组
B       1组
C       2组
D       2组
~~~



**测试coalesce，当分区数目减少的时候**

~~~scala
scala> val students = sc.parallelize(List("A","B","C","D","E","F"),3)
students: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[25] at parallelize at <console>:25

scala> students.coalesce(2).mapPartitionsWithIndex((index,paritions) =>{
     |   val stus = new ListBuffer[String]
     |   while (paritions.hasNext){
     |     stus += (paritions.next() + "\t" + (index + 1) + "组" )
     |   }
     |   stus.iterator
     | }).foreach(println)
C       2组
D       2组
E       2组
F       2组
A       1组
B       1组
~~~

**测试repartition，当分区增多的时候**

~~~scala
scala> val students = sc.parallelize(List("A","B","C","D","E","F"),3)
students: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[25] at parallelize at <console>:25

scala> students.repartition(4).mapPartitionsWithIndex((index,paritions) =>{
     |   val stus = new ListBuffer[String]
     |   while (paritions.hasNext){
     |     stus += (paritions.next() + "\t" + (index + 1) + "组" )
     |   }
     |   stus.iterator
     | }).foreach(println)
~~~



### 3. 结论：在生产上两者如何使用

repartition(numPartitions:Int):RDD[T]和coalesce(numPartitions:Int，shuffle:Boolean=false):RDD[T]

他们两个都是RDD的分区进行重新划分，repartition只是coalesce接口中shuffle为true的简易实现，（假设RDD有N个分区，需要重新划分成M个分区）

- 如果N<M。一般情况下N个分区有数据分布不均匀的状况，利用HashPartitioner函数将数据重新分区为M个，这时需要将shuffle设置为true。可以使用repartition或者coalesce为true的情况。

- 如果N>M并且N和M相差不多，那么就可以将N个分区中的若干个分区合并成一个新的分区，最终合并为M个分区，这时可以将shuffle设置为false，这时只能使用coalesce，在shuffle为false的情况下，如果M>N时，coalesce为无效的，不进行shuffle过程，父RDD和子RDD之间是窄依赖关系。

- 如果N>M并且两者相差悬殊，这时如果将shuffle设置为false，父子RDD是窄依赖关系，他们同处在一个Stage中，就可能造成spark程序的并行度不够，从而影响性能，如果在M为1的时候，为了使coalesce之前的操作有更好的并行度，可以将shuffle设置为true。

**总之：如果shuffle为false时，如果传入的参数大于现有的分区数目，RDD的分区数不变，也就是说不经过shuffle，是无法将RDDde分区数变多的。**

























