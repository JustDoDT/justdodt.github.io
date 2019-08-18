---
layout:     post
title:      "SparkSQL--从DataFrame说起"
date:       2019-08-01 01:28:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Spark

---



### SparkSQL 历史回顾

对SparkSQL了解的同学或多或少听说过Shark，不错，Shark就是SparkSQL的前身。2011的时候，Hive可以说是SQL On Hadoop的唯一选择，负责将SQL解析成MapReduce 任务运行在大数据上，实现交互式查询，报表登功能。就在这个时候，Spark社区的小伙伴意识到可以使用Spark作为执行引擎替换 Hive 中的 MapReduce ，这样可以使 Hive 的执行效率得到极大的提升。这个思想的产物就是 Shark，所以从实现功能上来看，Shark 更像一个 Hive On Spark 的实现版本。

改造完成刚开始，Shark 确实比 Hive 的执行效率有了极大的提升。然而，随着改造的深入，发现因为 Shark 继承了大量 Hive 代码导致添加优化规则等变得异常困难，优化的前景不再那么乐观。在意识到这个问题之后，Spark 社区经过一段时间激烈的思想斗争之后，还是毅然决然的在2014年彻底放弃 Shark ，转向 SparkSQL。

因此，可以理解为 SparkSQL 是一个全新的项目，接下来将会带大家一起走近 SparkSQL 的世界，从 SparkSQL体系的最顶端走向最底层，寻根问底，深入理解 SparkSQL 是如何工作的。



### SparkSQL 体系结构

SparkSQL 体系结构如图所示，整体由上到下分为三层：编程模型层，执行任务优化层以及任务执行引擎层，其中 SparkSQL 编程模型可以分为 SQL 和 DataFrame 两种；执行计划优化又称为 Catalyst，该模型负责将 SQL 语句解析成 AST (逻辑执行计划)，并对原始逻辑执行计划进行优化，优化规则分为基于规则的优化（Rule-based optimization, RBO）和基于代价的优化（Cost-based optimization, CBO）策略两种，最终输出优化后的物理执行计划；任务执行引擎就是 Spark 内核，负责根据物理执行计划生成 DAG ，在任务调度系统的管理下分解为任务集并分发到集群节点上加载数据运行，Tungsten基于对内存和CPU的性能优化，使得 Spark 能够更好地利用当前硬件条件提升性能，详细可以阅读 Spark 砖厂的  [Project Tungsten: Bringing Apache Spark Closer to Bare Metal](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html)

![spark](/img/Spark/SparkSQL/spark_df1.png)



SparkSQL 系列文章会按照体系结构由上至下的详细进行说明，本篇下面会重点讲解编程接口 DataFrame ，后面会利用 M 篇文章分析 Catalyst 工作原理，再后面会利用 N 篇文章分析 Spark 内核工作原理。



### SparkSQL 编程模型  -- DataFrame

说到计算模型，批处理计算从最初提出一直到现在，一共经历了两次大的变革，第一次大的变革是从 MapReduce 编程模式到 RDD 编程模型，第二次是从 RDD编程模型进化到 DataFrame模式。

#### 第一次变革：MR编程模型--> DAG编程模型

和 MR 编程模型相比，DAG 计算模型有很多改进：

> - `1.可以支持更多算子`，
>   - 比如 filter 算子，sum算子，reduceBykey等，不再像MR只支持map 和 reduce两种
> - `2.更加灵活的存储机制`，
>   - RDD 可以支持本地硬盘存储，缓存存储以及混合存储三种模式，用户可以进行选择。而MR目前只支持HDFS存储一种模式。很显然，HDFS存储需要将中间数据存储三份，而RDD则不需要，这是DAG编程效率高的一个重要原因之一。
> - `3.DAG模型带来了更细粒度的任务并发`，
>   - 不再像MR那样每次起个任务就要起个 JVM 进程，重死了；另外，DAG 模型带来了另一个利好是很好的容错性，一个任务即使中间断掉了，也不需要从头再来一次。
> - `4.延迟计算机制，`
>   - 一方面可以使得同一个 stage 内操作可以合并到一起落在一块数据上，而不再是所有数据先执行 a 操作，再扫描一遍执行 b 操作，太浪费时间。
>   - 另一个方面给执行路径优化留下了可能性，随便你怎么优化....

所有这些改进使得 DAG 编程模型相比 MR 编程模型，性能可以有 10~~100倍的提升！然而，DAG 计算模型就很完美吗？要知道，用户手写的 RDD 程序基本或多或少都会有些问题，性能也肯定不会是最优的。如果没有一个高手指点或者优化，性能依然有很大潜力。而且 RDD 编程模型易读性差。这就是促成了第二次变革，从 DAG 编程模型进化到 DataFrame 编程模型。

#### 第二次变革：DAG编程模型 -> DataFrame编程模型

相比RDD，DataFrame增加了 Schema 概念，从这个角度看，DataFrame有点类似于关系型数据库中表的概念。可以根据下图对比 RDD 与 DataFrame 数据结构的差别：

![spark](/img/Spark/SparkSQL/spark_df2.png)



直观上看，DataFrame 相比 RDD 多了一个表头，这个小小的变化带来了很多优化的空间：

> - 1.RDD 中每一行记录都是一个整体，因此你不知道内部数据组织形式，这就使得你对数据项的操作能力很弱。表现出来就是支持很少的而且是比较粗粒度的算子，比如 map 、fliter 算子等。而 DataFrame 将一行划分了多个列，每个列都有一定的数据格式，这与数据库表模式就很相似了，数据粒度相比更细，因此就能支持更多更细粒度的算子，比如 select 算子、groupby 算子、where 算子等。更重要的，后者的表达能力要远远强于前者，比如同一个功能用 RDD 和 DataFrame 实现：



![spark](/img/Spark/SparkSQL/spark_df3.png)



> - 2.DataFrame 的 Schema 的存在，数据项的转换也都将是类型安全的，这对于较为复杂的数据计算程序的调试是十分有利的，很多数据类型不匹配的问题都可以在编译阶段就被检查出来，而对于不合法的数据文件，DataFrame也具备一定的分辨能力。
> - 3.DataFrame Schema 的存在，开辟了另一种数据存储形式：列式数据存储。列式存储是相对于传统的行式存储而言的，简单来讲，就是将同一列的所有数据物理上存储在一起。对于列式存储和行式存储可以参考下图：

![spark](/img/Spark/SparkSQL/spark_df4.png)





**列式存储有2个重要的作用：**

- 首先，同一种类型的数据存储在一起可以很好的提升数据压缩效率，因为越“相似”的数据，越容易压缩。数据压缩可以减少空间需求，还可以减少数据传输过程中带宽需求，这对于类似于 Spark 之类的大内存计算引擎来讲，会带来极大的好处
- 另外，列式存储还可以有效减少查询过程中的实际 IO ,大数据领域很多 OLAP 查询业务通常只会检索部分列值，而不是粗暴的 select * ，这样列式存储可以有效执行 ‘’列裁剪“，将不需要查找额度列直接跳过。

> - 4.DAG 编程模式都是用户自己写的RDD 程序，自己写的或多或少是有性能的提升空间！而 DataFrame 编程模式集成了一个优化神奇 ---Catalyst，这玩意类似于 MySQL 的SQL 优化器，负责将用户写的 DataFrame 程序进行优化得到最优的执行计划，比如最常见的谓词下推。很显然，优化后的执行计算相比于手写的执行计划性能当然会好一些。下图是官方给出来的测试数据对比（测试过程是在10billion数据规模下进行过滤聚合）

![spark](/img/Spark/SparkSQL/spark_df5.png)



个人认为，RDD 和 DataFrame 的关系类似于汇编语言和 Java 语言的关系，同一个功能，如果你用汇编实现的话，一方面会写的很长，另一方面写的代码可能还不是最优的，可谓是又臭又长。而 Java 语言有很多高级语义，可以很方便的实现相关功能，另一方面是经过 JVM 优化后会更加高效。

### DataFrame VS Dataset

> - Dataset是分布式数据集合，她是Spark1.6中添加的一个新接口，她提供了 RDD 的又是（强类型，使用强大的 lambda函数的能力）和 Spark SQL优化执行引擎的优点。Dataset 可以从 JVM 对象构造，然后使用功能转换（map,flatMap,filter等）进行操作。Dataset API 可以用 Scala 和 Java。Python不支持 Dataset  API。但由于 Python 的动态特性，Dataset API 的许多好处已经可用（即您可以通过名称自然地访问行的字段row.columnName）。 R的情况类似。
>
> - DataFrame是一个组织成命名列的数据集。它在概念上等同于关系数据库中的表或R / Python中的数据框，但在底层具有更丰富的优化。 DataFrame可以从多种来源构建，例如：结构化数据文件，Hive中的表，外部数据库或现有RDD。 DataFrame API在Scala，Java，Python和R中可用。在Scala和Java中，DataFrame由行数据集表示。`在Scala API中，DataFrame只是Dataset [Row]的类型别名`。而在`Java API中，用户需要使用 Dataset <Row> 来表示DataFrame。`
>
>   我们经常将行的 Scala / Java Dataset 称为 DataFrame。





### 参考文献：

- [SparkSQL－从DataFrame说起](<http://hbasefly.com/2017/02/16/sparksql-dataframe/>)
- [SparkSQL官网](<http://spark.apache.org/docs/latest/sql-programming-guide.html>)





