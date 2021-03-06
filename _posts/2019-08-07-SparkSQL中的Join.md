---
layout:     post
title:      "SparkSQL中的Join"
date:       2019-08-07 01:28:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Spark

---



### 概述

Join是 SQL语句中的常用操作，良好的表结构能够将数据分散在不同的表中，使其符合某种范式，减少表冗余、更新容错等。而建立表和表之间关系的最佳方式就是Join操作。

`对于 Spark 来说有3中 Join 的实现，每种 Join 对应着不同的应用场景：`

>- 1.**Broadcast Hash Join ：适合一张较小的表和一张大表进行 join**
>
>- 2.**Shuffle Hash Join :  适合一张小表和一张大表进行 join，或者是两张小表之间的 join**
>- 3.**Sort Merge Join ：适合两张较大的表之间进行 join**



**前两者都基于的是 Hash Join，只不过在 hash join之前需要先 shuffle还是先 broadcast。下面将详细的解释一下这三种不同的 join的具体原理。**

###  **Join背景介绍**

Join是数据库查询永远绕不开的话题，传统查询SQL技术总体可以分为简单操作（过滤操作-where、排序操作-limit等），聚合操作 -groupBy等以及 Join操作等。其中Join操作是其中最复杂、代价最大的操作类型，也是 OLAP场景中使用相对较多的操作。因此很有必要聊聊这个话题。



另外，从业务层面来讲，用户在数仓建设的时候也会涉及Join使用的问题。通常情况下，数据仓库中的表一般会分为”低层次表”和“高层次表”。



所谓”低层次表”，就是数据源导入数仓之后直接生成的表，单表列值较少，一般可以明显归为维度表或者事实表，表和表之间大多存在外健依赖，所以查询起来会遇到大量 Join运算，查询效率相对比较差。而“高层次表”是在”低层次表”的基础上加工转换而来，通常做法是使用 SQL语句将需要 Join的表预先进行合并形成“宽表”，在宽表上的查询因为不需要执行大量 Join因而效率相对较高，很明显，宽表缺点是数据会有大量冗余，而且生成相对比较滞后，查询结果可能并不及时。



因此，为了获得实效性更高的查询结果，大多数场景还是需要进行复杂的 Join操作。Join操作之所以复杂，不仅仅因为通常情况下其时间空间复杂度高，更重要的是它有很多算法，在不同场景下需要选择特定算法才能获得最好的优化效果。关系型数据库也有关于Join的各种用法，姜承尧大神之前由浅入深地介绍过 MySQL Join的各种算法以及调优方案（关注公众号 InsideMySQL并回复 join可以查看相关文章）。本文接下来会介绍SparkSQL所支持的几种常见的 Join算法以及其适用场景。



### **Join常见分类以及基本实现机制**

当前 SparkSQL支持三种 Join算法－shuffle hash join、broadcast hash join以及 sort merge join。其中前两者归根到底都属于 hash join，只不过在 hash join之前需要先 shuffle还是先 broadcast。`其实，这些算法并不是什么新鲜玩意，都是数据库几十年前的老古董了（参考），只不过换上了分布式的皮而已。`不过话说回来，SparkSQL/Hive…等等，所有这些大数据技术哪一样不是来自于传统数据库技术，什么语法解析AST、基于规则优化（CRO）、基于代价优化（CBO）、列存，都来自于传统数据库。就拿 shuffle hash join和 broadcast hash join来说，hash join算法就来自于传统数据库，而 shuffle和 broadcast是大数据的皮，两者一结合就成了大数据的算法了。因此可以这样说，大数据的根就是传统数据库，传统数据库人才可以很快的转型到大数据。好吧，这些都是闲篇。



继续来看技术，既然 hash join是’内核’，那就刨出来看看，看完把’皮’再分析一下。



#### **Hash Join** 

先来看看这样一条 SQL语句：select * from order,item where item.id = order.i_id，很简单一个 Join节点，参与 join的两张表是 item 和 order，join key分别是 item.id 以及 order.i_id。现在假设这个 Join 采用的是 hash join 算法，整个过程会经历三步：

> - `1.确定 Build Table以及Probe Table：`这个概念比较重要，Build Table 使用 join key 构建 Hash Table，而 Probe Table 使用 join key 进行探测，探测成功就可以 join 在一起。通常情况下，小表会作为 Build Table，大表作为 Probe Table。此事例中 item为 Build Table，order为Probe Table。
>
> - ` 2.构建 Hash Table：`依次读取 Build Table（item）的数据，对于每一行数据根据 join key（item.id）进行 hash，hash 到对应的 Bucket，生成 hash table中的一条记录。数据缓存在内存中，如果内存放不下需要dump到外存。
>
> - `3.探测：`再依次扫描 Probe Table（order）的数据，使用相同的 hash函数映射 Hash Table中的记录，映射成功之后再检查 join条件（item.id = order.i_id），如果匹配成功就可以将两者 join在一起。



 ![spark](/img/Spark/SparkSQL/sparksql_join1.png)



**基本流程可以参考上图，这里有两个小问题需要关注：**

> - `1.hash join 性能如何？`很显然，hash join 基本都只扫描两表一次，可以认为 O(a+b)，较之最极端的笛卡尔集运算 a * b ，不知道甩了多少条街
> - `2.为什么 Build Table 选择小表？`道理很简单，因为构建的 Hash Table 最好能全部加载在内存，效率最高；这也决定了 hash join 算法只适合至少一个小表的 join 场景，对于两个大表的 join 场景并不适用；



上文说过，hash join 是传统数据库中的单机 join 算法，在分布式环境下需要经过一定的分布式改造，说到底就是尽可能利用分布式计算资源进行并行化计算，提高总体效率。**hash join 分布式改造一般有两种经典方案：**

>- `1.broadcast hash join: `将其中一张小表广播分发到另一张大表所在的分区节点上，分别并发地与其上的分区记录进行 hash join。broadcast 适用于小表很小，可以直接广播的场景。
>- `2.shuffle hash join: `一旦小表数据量较大，此时就不再适合进行广播分发。这种情况下，可以根据 join key 相同必然分区相同的原理，将两张表分别按照 join key 进行重新组织分区，这样就可以将 join 分而治之，划分为很多小 join，充分利用集群资源并行化。



#### **Broadcast Hash Join**

大家知道，在数据库的常见模型中（比如星型模型或者雪花模型），表一般分为两种：事实表和维度表。维度表一般指固定的、变动较少的表，例如联系人、物品种类等，一般数据有限。而事实表一般记录流水，比如销售清单等，通常随着时间的增长不断膨胀。

因为 Join 操作是对两个表中 key 值相同的记录进行连接，在 SparkSQL中，对两个表做Join最直接的方式是先根据 key 分区，再在每个分区中把 key 值相同的记录拿出来做连接操作。但这样就不可避免地涉及到 shuffle，而 shuffle 在 Spark中是比较耗时的操作，我们应该尽可能的设计 Spark 应用使其避免大量的 shuffle。

当维度表和事实表进行 Join 操作时，为了避免 shuffle，我们可以将大小有限的维度表的全部数据分发到每个节点上，供事实表使用。executor存储维度表的全部数据，一定程度上牺牲了空间，换取 shuffle 操作大量的耗时，这在 SparkSQL 中称作 Broadcast Join，如下图所示：



![spark](/img/Spark/SparkSQL/sparksql_join6.png)



Table B 是较小的表，黑色表示将其广播到每个 executor 节点上，Table A 的每个 partition 会通过 block manager 取到 Table A 的数据。根据每条记录的Join Key 取到 Table B 中相对应的记录，根据 Join Type 进行操作。这个过程比较简单，不做赘述。

**Broadcast Join的条件有以下几个：**

>- 1.被广播的表需要小于 spark.sql.autoBroadcastJoinThreshold 所配置的值，默认是10M （或者加了broadcast join的hint）
>- 2.基表不能被广播，比如 left outer join时，只能广播右表



看起来广播是一个比较理想的方案，但它有没有缺点呢？也很明显。这个方案只能用于广播较小的表，否则数据的冗余传输就远大于 shuffle 的开销；另外，广播时需要将被广播的表现 collect 到 driver端，当频繁有广播出现时，对driver的内存也是一个考验。



**如下图所示，broadcast hash join可以分为两步：**

>- `1.broadcast阶段：`将小表广播分发到大表所在的所有主机。广播算法可以有很多，最简单的是先发给 driver，driver再统一分发给所有 executor；要不就是基于 bittorrete的 p2p 思路；
>
>- `2.hash join阶段：`在每个 executor上执行单机版 hash join，小表映射，大表试探；



![spark](/img/Spark/SparkSQL/sparksql_join2.png)





> **`注意：`**
>
> SparkSQL规定 broadcast hash join 执行的基本条件为被广播小表必须小于参数 spark.sql.autoBroadcastJoinThreshold，默认为10M。



#### **Shuffle Hash Join**

当一侧的表比较小时，我们选择将其广播出去以避免 shuffle，提高性能。但因为被广播的表首先被 collect到 driver段，然后被冗余分发到每个 executor上，所以当表比较大时，采用 broadcast join 会对 driver 端和 executor 端造成较大的压力。

但由于 Spark 是一个分布式的计算引擎，可以通过分区的形式将大批量的数据划分成 n 份较小的数据集进行并行计算。这种思想应用到 Join 上便是 Shuffle Hash Join了。利用 key 相同必然分区相同的这个原理，**两个表中，key 相同的行都会被 shuffle 到同一个分区中，**SparkSQL 将较大表的 join 分而治之，先将表划分成 n 个分区，再对两个表中相对应分区的数据分别进行 Hash Join，这样即在一定程度上减少了 driver 广播一侧表的压力，也减少了 executor端取整张被广播表的内存消耗。其原理如下图：

![spark](/img/Spark/SparkSQL/sparksql_join7.png)

**Shuffle Hash Join分为两步：**

>- 1.对两张表分别按照 join keys 进行重分区，即 shuffle，目的是为了让有相同 join keys 值的记录分到对应的分区中
>
>- 2.对对应分区中的数据进行 join，此处先将小表分区构造为一张 hash表，然后根据大表分区中记录的 join keys值拿出来进行匹配



**Shuffle Hash Join的条件有以下几个：**

> - 1.分区的平均大小不超过 spark.sql.autoBroadcastJoinThreshold所配置的值，默认是10M 
>
> - 2.基表不能被广播，比如 left outer join时，只能广播右表
>
> -  3.一侧的表要明显小于另外一侧，小的一侧将被广播（明显小于的定义为3倍小，此处为经验值）



我们可以看到，在一定大小的表中，SparkSQL 从时空结合的角度来看，将两个表进行重新分区，并且对小表中的分区进行 hash 化，从而完成 join。在保持一定复杂度的基础上，尽量减少 driver 和 executor的内存压力，提升了计算时的稳定性。



在大数据条件下如果一张表很小，执行 join 操作最优的选择无疑是 broadcast hash join，效率最高。但是一旦小表数据量增大，广播所需内存、带宽等资源必然就会太大，broadcast hash join就不再是最优方案。此时可以按照 join key进行分区，根据key相同必然分区相同的原理，就可以将大表 join分而治之，划分为很多小表的 join，充分利用集群资源并行化。**如下图所示，shuffle hash join也可以分为两步：**

>- `1.shuffle 阶段：`分别将两个表按照 join key进行分区，将相同 join key 的记录重分布到同一节点，两张表的数据会被重分布到集群中所有节点。这个过程称为 shuffle
>
>- `2.hash join 阶段：`每个分区节点上的数据单独执行单机 hash join 算法。





![spark](/img/Spark/SparkSQL/sparksql_join3.png)





看到这里，可以初步总结出来如果两张小表 join 可以直接使用单机版 hash join；如果一张大表 join 一张极小表，可以选择 broadcast hash join 算法；而如果是一张大表 join 一张小表，则可以选择 shuffle hash join 算法；那如果是两张大表进行 join 呢？



#### **Sort-Merge Join**

上面介绍的两种实现对于一定大小的表比较适用，但当两个表都非常大时，显然无论适用哪种都会对计算内存造成很大压力。这是因为 join 时两者采取的都是 hash join，是将一侧的数据完全加载到内存中，使用 hash code 取 join keys 值相等的记录进行连接。

当两个表都非常大时，SparkSQL 采用了一种全新的方案来对表进行 Join，即 Sort Merge Join。这种实现方式不用将一侧数据全部加载后再进星 hash join，但需要在 join 前将数据排序，如下图所示：

![spark](/img/Spark/SparkSQL/sparksql_join8.png)





> 可以看到，首先将两张表按照 join keys 进行了重新 shuffle，保证 join keys 值相同的记录会被分在相应的分区。分区后对每个分区内的数据进行排序，排序后再对相应的分区内的记录进行连接，如下图示：

![spark](/img/Spark/SparkSQL/sparksql_join9.png)



**SparkSQL对两张大表join采用了全新的算法－sort-merge join，如下图所示，整个过程分为三个步骤：**

![spark](/img/Spark/SparkSQL/sparksql_join4.png)





> - `1.shuffle阶段：`将两张大表根据 join key 进行重新分区，两张表数据会分布到整个集群，以便分布式并行处理
>
> - `2.sort阶段：`对单个分区节点的两表数据，分别进行排序
>
> - `3.merge阶段：`对排好序的两张分区表数据执行 join 操作。join 操作很简单，分别遍历两个有序序列，碰到相同 join key 就 merge 输出，否则取更小一边，见下图示意：

![spark](/img/Spark/SparkSQL/sparksql_join5.png)



>`仔细分析的话会发现，sort-merge join 的代价并不比 shuffle hash join 小，反而是多了很多。那为什么 SparkSQL 还会在两张大表的场景下选择使用 sort-merge join 算法呢？这和 Spark 的 shuffle 实现有关，目前 spark 的 shuffle 实现都适用 sort-based shuffle 算法，因此在经过 shuffle 之后 partition 数据都是按照 key 排序的。因此理论上可以认为数据经过 shuffle 之后是不需要 sort 的，可以直接 merge。`



经过上文的分析，可以明确每种 Join 算法都有自己的适用场景，数据仓库设计时最好避免大表与大表的 join 查询，SparkSQL 也可以根据内存资源、带宽资源适量将参数 spark.sql.autoBroadcastJoinThreshold 调大，让更多 join 实际执行为 broadcast hash join。



### 总结

Join 操作是传统数据库中的一个高级特性，尤其对于当前 MySQL 数据库更是如此，原因很简单，MySQL 对 Join 的支持目前还比较有限，只支持 Nested-Loop Join 算法，因此在 OLAP 场景下 MySQL 是很难吃的消的，不要去用 MySQL 去跑任何 OLAP 业务，结果真的很难看。不过好消息是 MySQL 在新版本要开始支持 Hash Join 了，这样也许在将来也可以用 MySQL 来处理一些小规模的 OLAP 业务。

和 MySQL 相比，PostgreSQL、SQLServer、Oracle 等这些数据库对 Join 支持更加全面一些，都支持 Hash Join 算法。由 PostgreSQL 作为内核构建的分布式系统 Greenplum 更是在数据仓库中占有一席之地，这和 PostgreSQL对 Join 算法的支持其实有很大关系。

**总体而言，**传统数据库单机模式做 Join 的场景毕竟有限，也建议尽量减少使用 Join。然而大数据领域就完全不同，Join 是标配，OLAP 业务根本无法离开表与表之间的关联，对 Join 的支持成熟度一定程度上决定了系统的性能，夸张点说，`’得Join者得天下’。`本文只是试图带大家真正走进 Join 的世界，了解常用的几种 Join 算法以及各自的适用场景。后面两篇文章还会涉及 Join 的方方面面，敬请期待！



### 参考资料

- [SparkSQL -  有必要坐下来聊聊 Join ](<http://hbasefly.com/2017/03/19/sparksql-basic-join/>)
- [SparkSQL 3种Join实现](https://mp.weixin.qq.com/s/7tTU87rJ4TYfb6N2Rn2M5A)

