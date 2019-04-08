---
layout:     post
title:      "MapReduce分片"
date:       2018-04-09 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - hadoop, mapreduce
---

### 概述
在进行map计算之前，map会根据输入文件计算输入分片（input split）,每个输入分片（input split）针对一个map任务，输入分片（input split）存储的并非数据本身，而是一个分片长度和一个记录数据的位置的数组。

### 问题
Hadoop 2.x默认的block大小是128MB，Hadoop 1.x默认的block大小是64MB，可以在hdfs-site.xml中设置dfs.block.size，注意单位是byte。

分片大小范围可以在mapred-site.xml中设置，mapred.min.split.size mapred.max.split.size，minSplitSize大小默认为1B，maxSplitSize大小默认为Long.MAX_VALUE = 9223372036854775807

**那么分片到底是多大呢？**

minSize=max{minSplitSize,mapred.min.split.size} 

maxSize=mapred.max.split.size

splitSize=max{minSize,min{maxSize,blockSize}}

看一下源码：
[源码地址](https://github.com/apache/hadoop/blob/2addebb94f592e46b261e2edd9e95a82e83bd761/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-core/src/main/java/org/apache/hadoop/mapred/FileInputFormat.java)

    long blockSize = file.getBlockSize();
    long splitSize = computeSplitSize(goalSize, minSize, blockSize);
          
       .
       .
       .
    protected long computeSplitSize(long goalSize, long minSize,
                                       long blockSize) {
    return Math.max(minSize, Math.min(goalSize, blockSize));
          
   
   
所以在我们没有设置分片的范围的时候，分片大小是由block块大小决定的，和它的大小一样。比如把一个258MB的文件上传到HDFS上，假设block块大小是128MB，那么它就会被分成三个block块，与之对应产生三个split，所以最终会产生三个map task。我又发现了另一个问题，第三个block块里存的文件大小只有2MB，而它的block块大小是128MB，
**那它实际占用Linux file system的多大空间？**

答案是实际的文件大小，而非一个块的大小

**最后一个问题是**： 如果hdfs占用Linux file system的磁盘空间按实际文件大小算，那么这个”块大小“有必要存在吗？

其实块大小还是必要的，一个显而易见的作用就是当文件通过append操作不断增长的过程中，可以通过来block size决定何时split文件。

 
### 总结：
Split 分片的计算跟 blockSize, minSize, maxSize 三个参数有关
假如 blockSize 设置128 M
文件大小 200M
那么splitSize 就是128M
200/128=1.56>1.1 创建1个split （通过源码可以看出最右一个文件的大小在 0- 128+12.8M之间）
剩下72M在创建一个

   
   
   
  [源码地址](https://github.com/apache/hadoop/blob/2addebb94f592e46b261e2edd9e95a82e83bd761/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-core/src/main/java/org/apache/hadoop/mapred/FileInputFormat.java) 
   
   
          
          
