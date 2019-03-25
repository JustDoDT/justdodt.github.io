---
layout:     post
title:      "Spark RDD,MySQL,HDFS,Oracle的checkpoint之间的对比"
date:       2018-03-11 23:01:00
author:     "JustDoDT"
#header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - Spark
---



### Spark RDD 的checkpoint
检查点（本质是通过将RDD写入Disk做检查点）是为了通过lineage（血统）做容错的辅助，lineage过长会造成容错成本过高，这样就不如在中间阶段做检查点容错，如果之后有节点出现问题而丢失数据，从做检查点的RDD开始重做LIneage，就会减少开销。

设置checkpoint的目录，可以是本地文件夹、也可以是HDFS。一般是在具有容错能力，高可靠的文件系统上（比如HDFS、S3等）设置一个检查点路径，用于保存检查点数据。



#### 对比HDFS、MySQL、Oracle、Spark RDD的checkpoit

Snapshot（快照）：在数据库或者文件系统中，一个快照表示对当前系统状态的一个备份，当系统发生故障时，可以利用这个快照将系统恢复到产生快照时的样子。

Checkpoint（检查点）：因为数据库系统或者像HDFS这样的分布式文件系统，对文件的修改不是直接写到磁盘的，很多操作是先缓存到内存的Buffer中，当遇到一个检查点时，系统就强制将内存中的数据写回磁盘，当然此时才会记录日志，从而产生持久化的修改状态。

两者的区别：

Snapshot是对数据的备份，而Checkpoint只是一个将数据修改持久化的机制。

#### HDFS 的Checkpoint

Checkpoint主要干的事情是：

* 1.将NameNode中的edits和fsimages文件拷贝到Second NameNode上

* 2.然后将edits中的操作与fsimage文件merage以后形成一个新的fsimage，这样不仅完成了对现有NameNode数据的备份，而且还产生了持久化操作的fsimage。

* 3.Second NameNode需要把merage后的fsimage文件upload到NameNode上面，完成NameNode中的fsimage的更新。

#### MySQL的Checkpoint

Checkpoint的目的：

* 1.缩短数据库的恢复时间

* 2.buffer pool空间不够时，将脏页刷新到磁盘

* 3.redo log 不可用时，刷新脏页

#### Oracle的Checkpoint的作用

* 1.保证数据库的一致性，这是指将脏数据写入到硬盘，保证内存和硬盘上的数据时一样的

* 2.缩短实例恢复的时间，实例恢复要把实例异常关闭前没有写到硬盘的脏数据通过日志进行恢复。如果脏块过多，实例恢复的时间也会很长，检查点的发生可以减少脏块的数量，从而提高实例恢复的时间。
