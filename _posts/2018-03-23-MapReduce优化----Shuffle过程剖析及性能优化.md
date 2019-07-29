---
layout:     post
title:      "MapReduce优化----Shuffle过程剖析及性能优化"
date:       2018-03-23 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Hadoop
---



### 1. Map端

当Map 开始产生输出时，它并不是简单的把数据写到磁盘，因为频繁的磁盘操作会导致性能严重下降。它的处理过程更复杂，数据首先是写到内存中的一个缓冲区，并做了一些预排序，以提升效率。

每个Map 任务都有一个用来写入输出数据的循环内存缓冲区。这个缓冲区默认大小是100MB，可以通过io.sort.mb 属性来设置具体大小。当缓冲区中的数据量达到一个特定阀值(io.sort.mb * io.sort.spill.percent，其中io.sort.spill.percent 默认是0.80)时，系统将会启动一个后台线程把缓冲区中的内容spill 到磁盘。在spill 过程中，Map 的输出将会继续写入到缓冲区，但如果缓冲区已满，Map 就会被阻塞直到spill 完成。spill 线程在把缓冲区的数据写到磁盘前，会对它进行一个二次快速排序，首先根据数据所属的partition 排序，然后每个partition 中再按Key 排序。输出包括一个索引文件和数据文件。

如果设定了Combiner，将在排序输出的基础上运行。Combiner 就是一个Mini Reducer，它在执行Map 任务的节点本身运行，先对Map 的输出做一次简单Reduce，使得Map 的输出更紧凑，更少的数据会被写入磁盘和传送到Reducer。

spill 文件保存在由mapred.local.dir指定的目录中，Map 任务结束后删除。

每当内存中的数据达到spill 阀值的时候，都会产生一个新的spill 文件，所以在Map任务写完它的最后一个输出记录时，可能会有多个spill 文件。在Map 任务完成前，所有的spill 文件将会被归并排序为一个索引文件和数据文件。这是一个多路归并过程，最大归并路数由io.sort.factor 控制(默认是10)。如果设定了Combiner，并且spill文件的数量至少是3（由min.num.spills.for.combine 属性控制），那么Combiner 将在输出文件被写入磁盘前运行以压缩数据。

对写入到磁盘的数据进行压缩，通常是一个很好的方法，因为这样做使得数据写入磁盘的速度更快，节省磁盘空间，并减少需要传送到Reducer 的数据量。默认输出是不被压缩的， 但可以很简单的设置mapred.compress.map.output 为true 启用该功能。压缩所使用的库由mapred.map.output.compression.codec 来设定。

当spill 文件归并完毕后，Map 将删除所有的临时spill 文件，并告知TaskTracker 任务已完成。Reducers 通过HTTP来获取对应的数据。用来传输partitions 数据的工作线程数由tasktracker.http.threads 控制，这个设定是针对每一个TaskTracker 的，并不是单个Map，默认值为40，在运行大作业的大集群上可以增大以提升数据传输速率。

### 2. Reduce端

#### 2.1 copy阶段

Map 的输出文件放置在运行Map 任务的TaskTracker 的本地磁盘上（注意：Map 输出总是写到本地磁盘，但Reduce 输出不是，一般是写到HDFS），它是运行Reduce 任务的TaskTracker 所需要的输入数据。Reduce 任务的输入数据分布在集群内的多个Map 任务的输出中，Map 任务可能会在不同的时间内完成，只要完成的Map 任务数达到占总Map任务数一定比例（mapred.reduce.slowstart.completed.maps 默认0.05），Reduce 任务就开始拷贝它的输出。

         Reduce 任务拥有多个拷贝线程， 可以并行的获取Map 输出。可以通过设定mapred.reduce.parallel.copies 来改变线程数，默认是5。

如果Map 输出足够小，它们会被拷贝到Reduce TaskTracker 的内存中（缓冲区的大小

由mapred.job.shuffle.input.buffer.percent 控制，指定了用于此目的的堆内存的百分比）；如果缓冲区空间不足，会被拷贝到磁盘上。当内存中的缓冲区用量达到一定比例阀值（由mapred.job.shuffle.merge.percent 控制），或者达到了Map 输出的阀值大小（由mapred.inmem.merge.threshold 控制），缓冲区中的数据将会被归并然后spill到磁盘。

拷贝来的数据叠加在磁盘上，有一个后台线程会将它们归并为更大的排序文件，这样做节省了后期归并的时间。对于经过压缩的Map 输出，系统会自动把它们解压到内存方便对其执行归并。

#### 2.2 sort阶段

当所有的Map 输出都被拷贝后，Reduce 任务进入排序阶段（更恰当的说应该是归并阶段，因为排序在Map 端就已经完成），这个阶段会对所有的Map 输出进行归并排序，这个工作会重复多次才能完成。

假设这里有50 个Map 输出（可能有保存在内存中的），并且归并因子是10（由io.sort.factor 控制，就像Map 端的merge 一样），那最终需要5 次归并。每次归并会把10个文件归并为一个，最终生成5 个中间文件。

注：每趟合并的文件数实际上比示例中展示的更微妙。目标是合并最小数量的文件以便满足最后一趟的合并系数。因此如果是40个文件，我们不会在四趟中，每趟合并10个文件从而得到4个文件。相反，第一趟只合并4个文件，随后三趟合并所有十个文件。在最后一趟中，4个已合并的文件和余下的6个（未合并的）文件合计10个文件。这并没有改变合并的次数，它只是一个优化措施，尽量减少写到磁盘的数据量，因为最后一趟总是直接合并到reduce。

#### 2.3 reduce阶段

在Reduce 阶段，Reduce 函数会作用在排序输出的每一个key 上。这个阶段的输出被直接写到输出文件系统，一般是HDFS。在HDFS 中，因为TaskTracker 节点也运行着一个DataNode 进程，所以第一个块备份会直接写到本地磁盘。

### 3. 配置调优

该配置调优方案主要是对以上Shuffle整个过程中涉及到的配置项按流程顺序一一呈现并给以调优建议。

#### 3.1 Map端

- io.sort.mb
  - 用于map输出排序的内存缓冲区大小 
  - 类型：Int
  - 默认：100mb
  - 备注：如果能估算map输出大小，就可以合理设置该值来尽可能减少溢出写的次数，这对调优很有帮助。



- io.sort.spill.percent
  - map输出排序时的spill阀值（即使用比例达到该值时，将缓冲区中的内容spill 到磁盘）
  - 类型：float
  - 默认：0.80



- io.sort.factor
  - 归并因子（归并时的最多合并的流数），map、reduce阶段都要用到
  - 类型：Int
  - 默认：10
  - 备注：将此值增加到100是很常见的。



- min.num.spills.for.combine
  - 运行combiner所需的最少溢出写文件数（如果已指定combiner）
  - 类型：Int
  - 默认：3



- mapred.compress.map.output
  - map输出是否压缩
  - 类型：Boolean
  - 默认：false
  - 备注：如果map输出的数据量非常大，那么在写入磁盘时压缩数据往往是个很好的主意，因为这样会让写磁盘的速度更快，节约磁盘空间，并且减少传给reducer的数据量。



- mapred.map.output.compression.codec
  - 用于map输出的压缩编解码器
  - 类型：Classname
  - 默认：org.apache.Hadoop.io.compress.DefaultCodec
  - 备注：推荐使用LZO压缩。Intel内部测试表明，相比未压缩，使用LZO压缩的 TeraSort作业，运行时间减少60%，且明显快于Zlib压缩。



- tasktracker.http.threads
  - 每个tasktracker的工作线程数，用于将map输出到reducer。（注：这是集群范围的设置，不能由单个作业设置）
  - 类型：Int
  - 默认：40
  - 备注：tasktracker开http服务的线程数。用于reduce拉取map输出数据，大集群可以将其设为40~50。



#### 3.2 reduce端

- mapred.reduce.slowstart.completed.maps
  - 调用reduce之前，map必须完成的最少比例
  - 类型：float
  - 默认：0.05



- mapred.reduce.parallel.copies
  - reducer在copy阶段同时从mapper上拉取的文件数
  - 类型：int
  - 默认：5



- mapred.job.shuffle.input.buffer.percent
  - 在shuffle的复制阶段，分配给map输出的缓冲区占堆空间的百分比
  - 类型：float
  - 默认：0.70



- mapred.job.shuffle.merge.percent
  - map输出缓冲区（由mapred.job.shuffle.input.buffer.percent定义）使用比例阀值，当达到此阀值，缓冲区中的数据将会被归并然后spill 到磁盘。
  - 类型：float
  - 默认：0.66



- mapred.inmem.merge.threshold
  - map输出缓冲区中文件数
  - 类型：int
  - 默认：1000
  - 备注：0或小于0的数意味着没有阀值限制，溢出写将有mapred.job.shuffle.merge.percent单独控制。



- mapred.job.reduce.input.buffer.percent
  - 在reduce过程中，在内存中保存map输出的空间占整个堆空间的比例。
  - 类型：float
  - 默认：0.0
  - 备注：reduce阶段开始时，内存中的map输出大小不能大于该值。默认情况下，在reduce任务开始之前，所有的map输出都合并到磁盘上，以便为reducer提供尽可能多的内存。然而，如果reducer需要的内存较少，则可以增加此值来最小化访问磁盘的次数，以提高reduce性能。

 

#### 3.3 性能调优补充

相对于大批量的小文件，hadoop更合适处理少量的大文件。一个原因是FileInputFormat生成的InputSplit是一个文件或该文件的一部分。如果文件很小，并且文件数量很多，那么每次map任务只处理很少的输入数据，每次map操作都会造成额外的开销。


### 参考博客

[MapReduce之Shuffle过程详述](https://matt33.com/2016/03/02/hadoop-shuffle/)





