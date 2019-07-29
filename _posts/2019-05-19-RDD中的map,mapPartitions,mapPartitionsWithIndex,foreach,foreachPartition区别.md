---
layout:     post
title:      "RDD中的map,mapPartitions,mapPartitionsWithIndex,foreach,
		foreachPartition区别"
date:       2019-05-19 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Spark
---



### map

**首先查看源码描述**

~~~
/**
   * Return a new RDD by applying a function to all elements of this RDD.
   */
  def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }
~~~

`map是对所有元素进行函数处理，并且返回一个新的RDD`



### mapPartitions

**首先查看源码描述**

~~~
/**
   * Return a new RDD by applying a function to each partition of this RDD.
   *
   * `preservesPartitioning` indicates whether the input function preserves the partitioner, which
   * should be `false` unless this is a pair RDD and the input function doesn't modify the keys.
   */
  def mapPartitions[U: ClassTag](
      f: Iterator[T] => Iterator[U],
      preservesPartitioning: Boolean = false): RDD[U] = withScope {
    val cleanedF = sc.clean(f)
    new MapPartitionsRDD(
      this,
      (context: TaskContext, index: Int, iter: Iterator[T]) => cleanedF(iter),
      preservesPartitioning)
  }
~~~

`官网描述为应用一个函数作用于每一个分区并且返回一个新的RDD`

   

### mapPartitionsWithIndex

**源码描述**

~~~
/**
   * Return a new RDD by applying a function to each partition of this RDD, while tracking the index
   * of the original partition.
   *
   * `preservesPartitioning` indicates whether the input function preserves the partitioner, which
   * should be `false` unless this is a pair RDD and the input function doesn't modify the keys.
   */
  def mapPartitionsWithIndex[U: ClassTag](
      f: (Int, Iterator[T]) => Iterator[U],
      preservesPartitioning: Boolean = false): RDD[U] = withScope {
    val cleanedF = sc.clean(f)
    new MapPartitionsRDD(
      this,
      (context: TaskContext, index: Int, iter: Iterator[T]) => cleanedF(index, iter),
      preservesPartitioning)
  }
~~~



`源码解释为：通过将函数应用于此RDD的每个分区来返回新的RDD，同时跟踪原始分区的索引。`



### map VS mapPartition VS mapPartitionWithIndex

**测试代码**

```
# 假设一个rdd有10个元素，分成3个分区。如果使用map方法，map中的输入函数会被调用10次；而使用mapPartitions方法的话，其输入函数会只会被调用3次，每个分区调用1次。

scala>  val a = sc.parallelize(1 to 10, 3)
a: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[16] at parallelize at <console>:24

scala> def myfuncPerElement(e:Int):Int = {
     | 
     |            println("e="+e)
     | 
     |            e*2
     | 
     |       }
myfuncPerElement: (e: Int)Int

scala> def myfuncPerPartition ( iter : Iterator [Int] ) : Iterator [Int] = {
     | 
     |          println("run in partition")
     | 
     |          var res = for (e <- iter ) yield e*2
     | 
     |           res
     | 
     |     }
myfuncPerPartition: (iter: Iterator[Int])Iterator[Int]

scala> def myfunPerPartitionWithIndex(index:Int,iter:Iterator[Int]) ={
     |            println("run in partition"+"\t"+index)
     |            var res = for(e <- iter) yield e*2
     |          res
     |            }
myfunPerPartitionWithIndex: (index: Int, iter: Iterator[Int])Iterator[Int]
			  
scala>  val b = a.map(myfuncPerElement).collect
e=4
e=5
e=6
e=1
e=2
e=3
e=7
e=8
e=9
e=10
b: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)

scala> val c =  a.mapPartitions(myfuncPerPartition).collect
run in partition
run in partition
run in partition
c: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)

scala> val d = a.mapPartitionsWithIndex(myfunPerPartitionWithIndex).collect
run in partition        0
run in partition        1
run in partition        2
d: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
```



`从上面的例子可以看出，map是作用所有的元素，mapPartition是作用于每一个分区，虽然最后两者的结果是一样的，但是mapPartion的计算成本要小很多，Spark算子优化需要考虑这个算子，但是用mapPartion容易出现OOM，需要注意内存的使用情况。mapPartitionsWithIndex和mapPartitons类似，只是其参数多了个分区索引号。`



### foreach

**源码描述**

~~~
/**
   * Applies a function f to all elements of this RDD.
   */
  def foreach(f: T => Unit): Unit = withScope {
    val cleanF = sc.clean(f)
    sc.runJob(this, (iter: Iterator[T]) => iter.foreach(cleanF))
  }
~~~



`源码解释为，将函数f作用此RDD的所有元素，这是一个action算子。`



### foreachPartition

**源码解释**

~~~
/**
   * Applies a function f to each partition of this RDD.
   */
  def foreachPartition(f: Iterator[T] => Unit): Unit = withScope {
    val cleanF = sc.clean(f)
    sc.runJob(this, (iter: Iterator[T]) => cleanF(iter))
  }
~~~



`源码解释为，将函数f作用于此RDD的每个分区。她也是action算子。`



### foreach VS foreachPartition

**测试代码，使用accumulator与foreach结合**

~~~
scala> var cnt = sc.accumulator(0)
warning: there were two deprecation warnings; re-run with -deprecation for details
cnt: org.apache.spark.Accumulator[Int] = 0

scala>  var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[22] at makeRDD at <console>:24

scala> rdd1.foreach(x => cnt += x)

scala> cnt.value
res18: Int = 55

scala> rdd1.collect.foreach(println)
1
2
3
4
5
6
7
8
9
10
~~~



**foreachPartition测试**

~~~
scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[25] at makeRDD at <console>:24

scala> var allsize = sc.accumulator(0)
warning: there were two deprecation warnings; re-run with -deprecation for details
allsize: org.apache.spark.Accumulator[Int] = 0

scala> var rdd1 = sc.makeRDD(1 to 10,2)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[26] at makeRDD at <console>:24

scala> rdd1.foreachPartition { x => {
     | allsize += x.size
     | }}

scala> allsize.value
res27: Int = 10
~~~



### 总结

**mapPartions和mapPartionsWithIndex和foreachPartition都是对分区做处理，map和foreach是对每一个元素做处理；在Spark优化的时候，需要考虑对分区做处理的高级算子。但是对分区做处理的算子，还需要考虑内存，因为容易出现OOM。foreachPartiotion为action算子，搞作数据库的时候，尽量用对分区做处理的action算子，比如foreachPartion。**





















































