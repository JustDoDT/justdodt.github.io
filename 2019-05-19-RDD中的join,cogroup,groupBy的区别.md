---
layout:     post
title:      "RDD中的join,cogroup,groupBy的区别"
date:       2019-05-19 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - spark
---



### cogroup

**源码介绍**

~~~
/**
   * For each key k in `this` or `other1` or `other2` or `other3`,
   * return a resulting RDD that contains a tuple with the list of values
   * for that key in `this`, `other1`, `other2` and `other3`.
   */
  def cogroup[W1, W2, W3](other1: RDD[(K, W1)],
      other2: RDD[(K, W2)],
      other3: RDD[(K, W3)],
      partitioner: Partitioner)
      : RDD[(K, (Iterable[V], Iterable[W1], Iterable[W2], Iterable[W3]))] = self.withScope {
    if (partitioner.isInstanceOf[HashPartitioner] && keyClass.isArray) {
      throw new SparkException("HashPartitioner cannot partition array keys.")
    }
    val cg = new CoGroupedRDD[K](Seq(self, other1, other2, other3), partitioner)
    cg.mapValues { case Array(vs, w1s, w2s, w3s) =>
       (vs.asInstanceOf[Iterable[V]],
         w1s.asInstanceOf[Iterable[W1]],
         w2s.asInstanceOf[Iterable[W2]],
         w3s.asInstanceOf[Iterable[W3]])
    }
  }

~~~

`对于每一个key,在other1或者other2里面都可以，返回一个结果RDD，包含了一个元祖，元祖里面的每一个key，对应每一个other1、other2。`



### join

**源码介绍：**

~~~
/**
   * Return an RDD containing all pairs of elements with matching keys in `this` and `other`. Each
   * pair of elements will be returned as a (k, (v1, v2)) tuple, where (k, v1) is in `this` and
   * (k, v2) is in `other`. Performs a hash join across the cluster.
   */
  def join[W](other: RDD[(K, W)]): RDD[(K, (V, W))] = self.withScope {
    join(other, defaultPartitioner(self, other))
  }
~~~



`返回包含所有元素对的RDD，其中包含“this”和“other”中的匹配键。 每对元素将作为（k，（v1，v2））元组返回，其中（k，v1）在“this”中，而（k，v2）在“other”中。 在整个群集中执行散列连接。`

### cogroup VS join

**Join代码测试**

~~~
val rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
val rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)

scala> rdd1.join(rdd2).collect
res0: Array[(String, (String, String))] = Array((A,(1,a)), (C,(3,c)))
~~~

`将出现在最终结果中的相关联的所有键值。和关系型数据库的Inner Join类型。`



**cogroup测试**

~~~
val rdd1 = sc.makeRDD(Array(("A","1"),("B","2"),("C","3")),2)
val rdd2 = sc.makeRDD(Array(("A","a"),("C","c"),("D","d")),2)

scala> var rdd3 = rdd1.cogroup(rdd2).collect
res0: Array[(String, (Iterable[String], Iterable[String]))] = Array(
(B,(CompactBuffer(2),CompactBuffer())), 
(D,(CompactBuffer(),CompactBuffer(d))), 
(A,(CompactBuffer(1),CompactBuffer(a))), 
(C,(CompactBuffer(3),CompactBuffer(c)))
)
~~~



`一个键至少出现在两个RDD中的任何一个钟，它将出现在最终的结果中。这非常类似于关系型数据库的full outer join。`



### groupBy

**源码介绍**

~~~
/**
   * Return an RDD of grouped items. Each group consists of a key and a sequence of elements
   * mapping to that key. The ordering of elements within each group is not guaranteed, and
   * may even differ each time the resulting RDD is evaluated.
   *
   * @note This operation may be very expensive. If you are grouping in order to perform an
   * aggregation (such as a sum or average) over each key, using `PairRDDFunctions.aggregateByKey`
   * or `PairRDDFunctions.reduceByKey` will provide much better performance.
   */
  def groupBy[K](f: T => K)(implicit kt: ClassTag[K]): RDD[(K, Iterable[T])] = withScope {
    groupBy[K](f, defaultPartitioner(this))
  }
~~~



`返回分组项的RDD。 每个组由一个键和一系列映射到该键的元素组成。 不保证每个组内元素的排序，并且每次评估结果RDD时甚至可能不同。`

**测试代码**

~~~
scala> val a = sc.parallelize(1 to 9, 3)
a: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[29] at parallelize at <console>:24

scala> a.groupBy(x => { if (x % 2 == 0) "even" else "odd" }).collect
res29: Array[(String, Iterable[Int])] = Array((even,CompactBuffer(2, 4, 6, 8)), (odd,CompactBuffer(1, 3, 5, 7, 9))
~~~



### 总结

**cogroup 和join比较类似，cogroup相当于MySQL中的inner join；join相当于MySQL中的full outer join;而groupBy和他们不同，只是对key进行分组。**



### 参考

- [Spark算子介绍](http://homepage.cs.latrobe.edu.au/zhe/ZhenHeSparkRDDAPIExamples.html#cogroup)
- [Spark官网](http://spark.apache.org/docs/latest/rdd-programming-guide.html#CogroupLink)
- [Spark Basics (Labels)](http://apachesparkbook.blogspot.com/search/label/a74%7C%20cogroup%28%29)





























