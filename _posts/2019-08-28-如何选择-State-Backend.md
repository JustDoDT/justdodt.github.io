---
layout:     post
title:      "如何选择 State Backend"
date:       2019-08-28 02:28:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Flink



---



### 概述



本文将深入探讨有状态的流处理，更确切地说是 Apache Flink 中不同的状态后端（State Backend）。在以下部分，我们将介绍 Apache Flink 的 3 种状态后端，它们的局限性以及根据具体案例需求选择最合适的状态后端。

在有状态的流处理中，当开发人员启用了 Flink 中的 checkpoint 机制，那么状态将会持久化以防止数据的丢失并确保发生故障时能够完全恢复。选择何种后端，将决定状态持久化的方式和位置。

Flink 提供了三种可用的状态后端：`MemoryStateBackend` ，`FsStateBackend` ，`RocksDBStateBackend`  。

![flink]()





### MemoryStateBackend

`MemoryStateBackend`  是将状态维护在 Java 堆上的一个内部状态后端。键值状态和窗口算子使用哈希表来存储数据（values）和定时器（timers）。当应用程序 checkpoint 时，此后端会在将状态发给 JobManager 之前快照下状态，JobManager 也将状态存储在 Java 堆上。默认情况下，MemoryStateBackend 配置成支持异步快照。**异步快照可以`避免阻塞数据流`的处理，从而`避免反压`的发生。**



**使用 MemoryStateBackend 时的注意点：**

- 默认情况下，每一个状态的大小限制为 5 MB 。可以通过 MemoryStateBackend 的构造函数增加这个大小。
- 状态大小受到 akka 帧大小的限制，所以无论怎么调整状态大小配置，都不能大于 akka 的帧大小。也可以通过 `akka.framesize`  调整 akka 帧大小（通过[配置文档](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/config.html)了解更多）。

- 状态的总大小不能超过 JobManager 的内存。

**何时使用 MemoryStateBackend：**

- 本地开发或调试时建议使用 `MemoryStateBackend` ，因为这种场景的状态大小是有限的
- `MemoryStateBackend` 最适合小状态的应用场景。例如 [Kafka consumer](<https://www.ververica.com/blog/kafka-flink-a-practical-how-to>)，或者一次仅一记录的函数 （Map, FlatMap，或 Filter）。

### FsStateBackend

FsStateBackend 需要配置的主要是文件系统，如 URL (类型，地址，路径)。举个例子，比如可以是：

- "hdfs://namenode:40010/flink/checkpoints"  或者
- "s3://flink/checkpoints"

当选择使用 FsStateBackend 时，正在进行的数据会被存在 TaskManager 的内存中。在 checkpoint 时，此后端会将状态快照写入配置的文件系统和目录的文件中，同时会在 JobManager 的内存中（在高可用场景下会存在 Zookeeper 中）存储极少的元数据。

默认情况下，FsStateBackend 配置成提供异步快照，以避免在状态 checkpoint 时阻塞数据流的处理。该特性可以实例化 FsStateBackend 时传入 false 的 布尔标志来禁用掉，例如：

> ```
> new FsStateBackend(path, false);
> ```



**使用 FsStateBackend 时的注意点：**

- 当前的状态仍然会先存在 TaskManager 中，所以状态的大小不能超过 TaskManager 的内存

**何时使用 FsStateBackend:**

- FsStateBackend 适用于处理大状态，长窗口，或大键值状态的有状态处理任务。
- FsStateBackend 非常适合用于高可用方案。



### RocksDBStateBackend

RocksDBStateBackend 的`配置也需要一个文件系统`（类型，地址，路径），如下所示：

- “hdfs://namenode:40010/flink/checkpoints” 或者
- “s3://flink/checkpoints”

`RocksDB 是一种嵌入式的本地数据库。`RocksDBStateBackend 将处理中的数据使用 RocksDB 存储在本地磁盘上。在 checkpoint 时，整个 RocksDB 数据库会被存储到配置的文件系统中，或者在超大状态作业时可以将增量的数据存储到配置的文件系统中。同时 Flink 会将极少的元数据存储在 JobManager 的内存中，或者在 Zookeeper 中（对于高可用的情况）。RocksDB 默认也是配置成异步快照的模式。

**使用 RocksDBStateBackend 时的注意点：**

- RocksDB 支持的单 key 和单 value 的大小最大为每个 2^31 字节。这是因为 RocksDB 的 JNI API 是基于 byte[] 的。
- 我们需要强调的是，对于使用具有合并操作的状态的应用程序，例如 ListState，随着时间可能会累积到超过 2^31 字节大小，这将会导致在接下来的查询中失败。

**何时使用 RocksDBStateBackend：**

- RocksDBStateBackend 最适合用于处理大状态，长窗口，或大键值状态的有状态处理任务。
- RocksDBStateBackend 非常适合用于高可用方案。
- RocksDBStateBackend 是目前唯一支持增量 checkpoint 的后端。增量 checkpoint 非常使用于超大状态的场景。

当使用 RocksDB 时，状态大小只受限于磁盘可用空间的大小。这也使得 RocksDBStateBackend 成为管理超大状态的最佳选择。使用 RocksDB 的权衡点在于所有的状态相关的操作都需要序列化（或反序列化）才能跨越 JNI 边界。与上面提到的堆上后端相比，这可能会影响应用程序的吞吐量。

不同状态后端满足不同场景的需求，在开始开发应用程序之前应该仔细考虑和规划后选择。这可确保选择了正确的状态后端以最好地满足应用程序和业务需求。





### 参考资料

- [本博客的原文地址](<https://data-artisans.com/blog/stateful-stream-processing-apache-flink-state-backends>)

- [Apache Flink配置文档](<https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/config.html>)

  

