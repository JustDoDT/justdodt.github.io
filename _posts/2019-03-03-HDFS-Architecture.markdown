---
layout:     post
title:      "HDFS 架构与工作原理"
date:       2019-03-03 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - hadoop, hdfs
---



# 一、目标
HDFS 全称 Hadoop 分布式文件系统，其最主要的作用是作为 Hadoop 生态中各系统的存储服务。

![Hadoop 生态](/img/2018-04-05-hdfs-Architecture/1.png)

面对大规模的数据，HDFS 在设计上满足了以下目标：
- 高度容错性：HDFS 可能由成百上千的服务器构成，任何一个组件都可能失效，因此错误检测和快速、自动的恢复时 HDFS 最核心的架构目标。
- 支持大规模数据集：运行在 HDFS 应用具有很大的数据集，它应该能提供整体上高的数据传输带宽，并能支撑数以千万计的文件。
- 支持流式读取数据：HDFS 的设计更多的考虑到了数据批处理，而不是用户交互处理，比之数据访问延迟的问题，更关键的是数据访问的高吞吐量。
- 简单的一致性模型：“一次写入多次读取”的文件访问模型简化了数据一致性的问题，并且是高吞吐量称为可能。
- 移动计算而非移动数据：一个应用的请求，离它操作的数据越近就越高效，HDFS 提供了将它们自己移动到数据附近的接口。
- 异构软硬件平台间的可移植性：平台的可移植性，方便用户也方便 HDFS 作为大规模数据应用平台的推广。

那么通过以上目标，HDFS 被设计成了什么样的架构呢？

# 二、架构与原理

## 1. HDFS 服务

根据 Cloudera Manager 安装 hadoop 2.6.0-cdh5.8.3 版本后，得到如下的 HDFS 服务：

![HDFS 服务](/img/2018-04-05-hdfs-Architecture/2.png)
 
如上图所示，HDFS 使用单一的 NameNode 节点简化了整体的设计，同时使用 Master-Slave 模式，防止 NameNode 成为单点故障，Failover Controller（故障切换器）的工作便是负责监控 NameNode 的状态与切换主从服务器。与此同时，为了能够快速从故障中恢复，每一次的数据读写删操作都会记录在 NameNode 上的 EditLog 中并同步到每个 JournalNode 节点。而 DataNode 节点则负责存储物理数据，为了应对不确定的故障，每一份数据默认被保存为 3 份，并分散在不同的 DataNode 中，而通过 Balancer 则可以平衡集群之间各节点的磁盘利用率，以防止某一个 DataNode 节点存储已满但是其它 DataNode 节点却可能为空的情况。
  
最后为了方便用户操作，HDFS 提供了 HttpFS 服务，用以通过 HTTP 方式访问 HDFS 服务的功能。默认的，你可以通过 http://[master namenode host]:50070/ 访问这个功能。
  
总的来说，HDFS 主要包含了 6 个服务，它们主要的功能如下：
- NameNode：负责管理文件系统的 namespace 以及客户端对文件的访问；
- DataNode：用于管理它所在节点上的存储；
- FailoverController：故障切换控制器，负责监控与切换 Namenode 服务；
- JournalNode：用于存储 EditLog；
- Balancer：用于平衡集群之间各节点的磁盘利用率；
- HttpFS：提供 HTTP 方式访问 HDFS 的功能。

通常而言，在关注 HDFS 架构时，总是关注 Namenode 和 Datanode 的架构，因为它们是 HDFS 的核心，也是客户端操作数据需要依赖的两个服务，所以再来看看 Namenode & Datanode 的架构吧。

## 2. NameNode & DataNode

![NameNode & DataNode 架构](/img/2018-04-05-hdfs-Architecture/3.png)

在 HDFS 中，Namenode 是 HDFS 中的主节点，用于维护和管理 Datanode 上存在的 block。它是一个高度可用的服务器，用于管理文件的 namespace 并控制客户端对文件的访问。HDFS 体系的构建方式是，用户数据永远不会驻留在 Namenode 上，数据只会驻留在 Datanode 上。

Namenode 的功能：
- 它是维护和管理 Datanode 的主守护进程；
- 它记录存储在集群中的所有文件的元数据，例如 block 的位置、文件大小、权限、层次结构等。有两个文件与元数据关联：
	- FsImage：它包含自 Namenode 开始以来文件的 namespace 的完整状态；
	- EditLogs：它包含最近对文件系统进行的与最新 FsImage 相关的所有修改。
- 它记录了发生在文件系统元数据上的每个更改。例如，如果一个文件在 HDFS 中被删除，Namenode 会立即在 EditLog 中记录这个操作。
- 它定期从集群中的所有 Datanode 接收心跳信息和 block 报告，以确保 Datanode 处于活动状态；
- 它保留了 HDFS 中所有 block 的记录以及这些 block 所在的节点；
- 它负责管理所有 block 的复制；
- 在 Datanode 失败的情况下，Namenode 会为副本选择新的 Datanode，平衡磁盘使用并管理到 Datanode 的通信流量。

Datanode 则是 HDFS 中的从节点，与 Namenode 不同的是，Datanode 是一种商品硬件，它并不具有高质量或高可用性。Datanode 是一个将数据存储在本地文件 ext3 或 ext4 中的 block 服务器。

Datanode 功能：
- 这些是丛属守护进行或在每台从属机器上运行的进程；
- 实际的数据存储在 Datanode 上；
- 执行文件系统客户端底层的读写请求；
- 定期向 Namenode 发送心跳报告 HDFS 的整体健康状况，默认频率为 3 秒。

上面提到 HDFS 中的数据是以 block 的形式分散在 DataNode 中，那什么是 block ，它是如何形成的呢？

## 3. 数据块（Block）

通常，在任何文件系统中，都将数据存储为 block 的集合。block 是硬盘上存储数据的最不连续的位置。在 hadoop 集群中，每个 block 的默认大小为 128M（此处指 hadoop 2.x 版本，hadoop 1.x 版本为 64M），您也可以通过如下配置配置 block 的大小：
```
dfs.block.size 或 dfs.blocksize = 64M
```

![example.txt](/img/2018-04-05-hdfs-Architecture/4.png)
 
HDFS 不会将每个文件存储在配置的 block 大小的确切倍数中，比如一个 514M 的文件 example.txt，如果上图所示，假设 block 大小为默认的 128M，那么将会创建 5 个block，前 4 个 block 大小为 128M，但是最后一个 block 的大小则仅为 2M。


那么，为何需要将 block 的大小设置为如此大，比如 128M，而不是更小呢？

通常我们在谈论 HDFS 的作用的时，都会谈论到巨大的数据集，即 Terabytes 和 PB 数据，所以如果我们的 block 大小仅为 4KB，那么将会产生太多的 block，间接导致产生太多的元数据，从而使管理这些 block 和 元数据会产生巨大的开销，这样无疑会增加 Namenode 和 Datanode 的负载，尤其 Namenode 是一个中心服务器，所以这并不会是我们想要的。

## 4. 数据复制

HDFS 提供了一种将大数据作为数据块存储在分布式环境中的可靠方法，即将这些 block 复制以容错。默认的复制因子是 3，您也可以通过如下配置配置复制因子：
```
fs.replication = 3
```

因此，如下图所示，每个 block 被复制 3 次存储在不同的 Datanode 中。

![数据复制](/img/2018-04-05-hdfs-Architecture/5.png)

所以，如果使用默认配置在 HDFS 中存储 128M 的文件，则最终将占用 384M （3 * 128M）的空间，因为这些 block 将被复制 3 次，并且每个副本将驻留在不同的 Datanode 中。

注意：Namenode 会定期的 从 Datanode 中收集 block 报告以维护复制因子。因此，无论何时 block 被过度复制或复制不足，Namenode 都会根据需要删除或添加副本。

## 5. 元数据

我们知道，Namenode 对我们来说相当的重要，如果它失败了，我们注定要失败。不过 HDFS 有对它做高可用的解决方案，高可用的方案中，如何同步状态是一个关键，所以这里再介绍一下那些保存在 Namenode 上的元数据。

注：以下元数据同步的方式使用的是通过JournalNode 节点同步的方式。

NameNode 将整个 namespace ，包括 block 到文件的映射、文件的属性等，都存储在一个称为 FsImage 的文件中，它被存放在内存以及本地文件系统中。而任何对于 namespace 元数据产生修改的操作，NameNode 都会使用一种称为 EditLog 的事务日志记录下来。例如在 HDFS 中创建一个文件，NameNode 就会在 EditLog 中插入一条记录来表示；同样的，修改文件的副本系数也将往 EditLog 插入一条记录。主 NameNode 会在本地文件系统中存储这个 EditLog，并同步到各个 JournalNode 节点上，而从 NameNode 则通过在 JournalNode 节点中读取 EditLog 来完成同步。

![元数据复制](/img/2018-04-05-hdfs-Architecture/6.png)

当 NameNode 启动时，它会从硬盘中读取 EditLog 和 FsImage，将所有 EditLog 中的事务作用在内存中的 FsImage 上，并将这个新版本的 FsImage 从内存中保存到本地磁盘上，然后删除旧的 EditLog，这个过程也被称为一个 checkpoint。

那么通过 NameNode 上的元数据可以确定 block 到具体 DataNode 节点的一个映射，所以客户端在读取或者写入数据到 HDFS 时，都是先到 NameNode 上获取元数据，然后根据元数据中的地址直接与 DataNode 交互，与此同时，客户端会缓存一段时间的元数据，从而减少与 NameNode 的交互。

那么一个完整的读取和写入流程是怎样的呢？

## 6. 数据写入原理

假如 HDFS 客户端想要写入一个名为 "example.txt" 的大小为 128MB 的文件。

![example.txt 分裂](/img/2018-04-05-hdfs-Architecture/7.png)

假定系统 block 的配置大小为 128MB（默认），那么客户端将把文件 "example.txt" 分成 2 个 block - 128 MB（block A） + 128MB（block B）。

接下来，每当客户端将数据写入 HDFS 时，将遵循以下协议：
- 首先，HDFS 客户端将与 NameNode 联系以获得针对两个 block（例如 block A 和 block B）的写入请求；
- 然后，NameNode 将授予客户端写入权限，并将提供最终将复制文件的 DataNode 的 IP 地址。
- 根据 HDFS 的可用性，复制因素和机架感知，DataNode IP 地址的选择纯碎是随机的。
- 假设复制因子被设置成默认值 3，那么对于每个 block，NameNode 将向客户端提供 3 个 DataNode 的 IP 地址列表。该列表对于每个 block 都是唯一的。假设分配结果如下：
- 对于 block A，列表 A = { DN 1 IP， DN 4 IP, DN 6 IP }
- 对于 block B，列表 B =｛DN 3 IP， DN 7 IP, DN 9 IP｝
- 每个 block 将被复制到 3 个不同的 DataNode 中，以保持整个集群中复制因子的一致。
- 接下来，整个数据的复制将分为 3 个阶段进行：
	1) 管道设置
	2) 数据流和复制
	3) 管道关闭与确认阶段
	
![复制阶段](/img/2018-04-05-hdfs-Architecture/8.png)

### 1) 管道设置

在写入 block 之前，客户端确认每个 IP 列表中的 DataNode 是否准备好接受数据，这样做时，客户端会通过连接该 block 列表中的各个 DataNode，为每个 block 创建一个管道。比如 block A，它提供的 DataNode 列表是：
列表 A = { DN 1 IP， DN 4 IP, DN 6 IP }

![管道设置](/img/2018-04-05-hdfs-Architecture/9.png)

因此，对于 block A，客户端将创建以下步骤来创建管道：
- 客户端将选择列表中的第一个 DataNode （DN1）并建立 TCP/IP 连接；
- 客户端将通知 DN 1 准备好接收该 block，它还会将下两个 DataNode（DN 4, 6）的 IP 地址提供给 DN 1；
- DN 1 将连接 DN 4，并通知 DN 4 准备好接受该 block，同时将 DN 6 的 IP 地址告知给 DN 4。然后 DN 4 将告诉 DN 6 准备好接受数据；
- 接下来，准备就绪的确认将遵循相反的顺序，即从 DN 6 -> DB 4 -> DN 1；
- 最后 DN 1 将通知客户端所有的 DataNode 都以准备就绪，并且将在 DataNode 1，4 和 6 之间形成管道；
- 现在，管道设置完成，客户端将最终的数据以流式方式处理。

### 2) 数据流

由于管道已经创建，客户端会将数据推送到管道中。不过不要忘记，在 HDFS 中，数据是基于复制因子进行复制的，所以，在假设复制因子为 3 时，block A 将存储到 3 个 DataNode 中。移动到最前的，客户端仅仅是将 block A 复制到 DN1。复制总是按照顺序进行的。

![数据流](/img/2018-04-05-hdfs-Architecture/10.png)

因此，在复制过程中将执行以下的步骤：
- 一旦该 block 被客户端写入 DN 1，DN 1 将连接到 DN 4；
- 然后，DN 1 将推送管道中的 block，数据将被复制到 DN 4；
- 同样的 DN 4 将连接到 DN 6 并将 block 复制为最后一个副本。

### 3) 管道关闭与确认阶段

一旦 block 被复制到所有的 3 个 DataNode 中，就会发生一系列的确认操作，以确保客户端和 NameNode 确信数据已经写入成功。然后客户端将最终关闭管道以结束 TCP 会话。

如下图所示，确认以相反的顺序发生，即从 DN 6 到 DN 4，再到 DN 1。最后，DN 1 将把 3 个确认（包括它自己的）推送到流水线中并发送给客户端，客户端将通知 NameNode 数据已经成功写入。此时，NameNode 将更新元数据，客户端将关闭管道。

![管道关闭与确认阶段](/img/2018-04-05-hdfs-Architecture/11.png)

类似的，block B 也将被复制到与 block B 并行的 DataNode 中，因此，这里需要注意一下几点：
- 客户端将同时将 block A 和 block B 复制到第一个 DataNode 上；
- 因此，在这个示例下，将为两个 block 形成两条管道，上述所有过程将在这两天管道中并行发生；
- 客户端将该 block 写入第一个 DataNode，然后 DataNode 将顺序复制该 block

![多文件复制](/img/2018-04-05-hdfs-Architecture/12.png)

如上图所示，客户端为两个 block 一共创建了两个管道，以下是各个管道中每个 block 正在进行的操作流程：
- block A 的管道：1A -> 2A -> 3A -> 4A
- block B 的管道：1B -> 2B -> 3B -> 4B -> 5B -> 6B

## 7. 数据写入实现

![数据写入实现](/img/2018-04-05-hdfs-Architecture/13.png)

写入数据的详细流程：
1) 客户端通过在 DistributedFileSystem 上调用 create() 方法来创建文件（步骤1）；  
2) DistributedFileSystem 对 NameNode 进行 RPC 调用，以在文件系统的 namespace 中创建一个新的文件，此时没有与之关联的 block（步骤2）（NameNode 会执行各种检查以确保文件不存在，并且确保客户端具有创建文件的正确权限。只有通过了这些检查，才会创建新文件成功，否则客户端抛出 IOException）  
3) DistributedFileSystem 返回一个 FSDataOutputStream 对象以开始写入数据。与读取数据一样，FSDataOutputStream 封装了一个 DFSOutputStream 对象，它处理与DataNode 和 NameNode 的通信。当客户端开始写入数据（步骤3）时，DFSOutputStream 将其拆分成数据包（packet），写入内部的数据队列，数据队列由 DataStreamer 使用，它通过选择合适的 DataNode 列表来存储副本，从而要求 NameNode 分配新的 block。DataNode列表会形成一个管道（假设副本数为3），其中包含3个节点。  
4) DataStreamer 将数据包以流式传输的方式传输到流水线中的第一个 DataNode，该数据流将数据包存储到第一个 DataNode 中并将其转发到流水线中的第二个 DataNode。类似地，第二个 DataNode 节点会将数据包转发到流水线中的第三个 DataNode 节点（步骤4）；  
5) DFSOutputStream 还维护了一个正在等待的数据包的内部的 ack 队列，由 DataNode 确认。只有在流水线中的书友 DataNode 节点都确认了数据包（步骤5）后才会将数据包从 ack 队列中删除；  
6) 如果数据在写入过程中发生故障，那么：(1) 首先关闭管道，并将 ack 队列中的所有数据包添加到数据队列的前端，以便故障节点下游的 DataNode 不会错过任何数据包。(2) 正常状态的 DataNode 上的 block 会被赋予一个新的标识，以便如果失败的 DataNode 稍后恢复后，删除发生故障的 DataNode 上的部分 block。(3) 然后将失败的 DataNode 从流水线中移除，并将该 block 的其余数据写入流水线中的两个良好的 DataNode。(4) 当 NameNode 注意到该 block 被复制不足时，会安排它在另外一个节点上创建另一个副本。  
如果多个 DataNode 在写入 block 发生故障，那么只要写入 dfs.replication.min 最小副本数，写入操作也会成功，该 block 将异步复制，知道其目标复制因子达到 dfs.replication 指定的数量。  
7) 当客户端完成写入数据后，它会在流上调用 close() 方法（步骤6）。  
8) close() 操作会将所有剩余的数据包刷新到 DataNode 管道，并联系 NameNode 以表示文件以传输完成（步骤7），并等待确认。  
9) NameNode 已经知道该文件由哪些 block 组成（因为通过 DataStreamer 分配的 block），所以它只需要等待 block 最小程度（dfs.replication.min）的被复制，便可以返回成功，也是此时，NameNode 才会将文件创建操作提交到 EditLog 中。  

## 8. 数据读取原理

HDFS 读取原理比较简单，参考上面的例子，假如 HDFS 客户端现在想要读取“example.txt“。

![数据读取原理](/img/2018-04-05-hdfs-Architecture/14.png)

现在，读取数据将发生以下步骤：
- 客户端将与 NameNode 联系，询问文件”example.txt“的 block 元数据；
- NameNode 返回存储的每个块（block A 和 block B）的 DataNode 列表；
- 然后，客户端将连接到列表中最近的 DataNode；
- 客户端开始从 DataNode 并行读取数据（DN 1 的 block A 和 DN 3 的 block B）
- 一旦客户端获得了所有必须的 block，它就会将这些 block 组合起来形成一个文件。
 

在提供给客户端读取请求时，HDFS 选择最接近客户端的副本，这减少了读取延迟和带宽消耗。因此，如果可能，会选择与阅读节点位于同一个机架上的副本。

## 9. 数据读取实现

![数据读取实现](/img/2018-04-05-hdfs-Architecture/15.png)

读取数据的详细流程：  
1) 客户端通过调用 FileSystem 对象的 open() 方法来打开它希望读取的文件，其实就是创建了一个 DistributedFileSystem 对象（步骤1）；  
2) DistributedFileSystem 使用 RPC 调用 NameNode 来确定文件中前几个 block 的位置，同时对于每个 block，NameNode 返回具有该 block 副本的 DataNode 的地址，此外，DataNode 根据它们与客户端的接近程度进行排序（根据集群网络的拓扑结构），如果客户端本身就是一个 DataNode，那么它将从本地直接读取（步骤2）。  
3) DistributedFileSystem 返回一个 FSDataInputStream（支持文件搜索的输入流）给客户端，供其从中读取数据；  
4) FSDataInputStream 依次包装一个 DFSInputStream，它们用于管理 DataNode 和 NameNode 的 I/O 读写；  
5) 客户端调用 stream 上的 read() 方法（步骤3）；  
6) DFSInputStream 中存储了文件中前几个 block 所在 DataNode 的地址，根据这些信息连接到文件中离 block 的最近的DataNode。数据从 DataNode 返回至客户端，客户端在数据流上重复调用 read() 方法（不同 block 并行读取）（步骤4，5）；  
7) 当 block 全部传输完成后，DFSInputStream 将关闭与 DataNode 的连接；  
8) 当客户端完成数据读取后，调用 FSDataInputStream 上 close() 方法以关闭流（步骤6）；  
心跳检测、block 状态报告与数据重新复制  
文件以 block 形式写入 DataNode 后，其副本数必须满足系统配置的数量（dfs.replication），即使是在之后 DataNode 发生了故障，比如磁盘错误或者宕机，HDFS 都应该有能力来处理这样的情况。所以这里就涉及到了几个问题：  
1) HDFS 如果确定 Datanode 的状态  
2) 如何确定哪些 block 出现了丢失  
3) Datanode 发生故障后，如何保障数据的安全  

在 HDFS 中，Datanode 以及 block 的元信息都通过 Namenode 来管理。Namenode 会周期性地从集群中的每个 Datanode 接收心跳信号和 block 状态报告（Blockreport）。接收到心跳信号意味着该 Datanode 节点工作正常，而 block 状态报告则包含了一个该 Datanode 上所有 block 的列表。  

那么根据心跳信号以及 block 状态报告，Namenode 可以知道每一个 Datanode 是否正常工作，以及哪些 block 被损坏了。如果一个 Datanode 宕机了，那么任何存储在它之上的所有 block 将不再有效。Namenode 不断地检测这些 block 是否满足副本系数，一旦发现某个 block 的副本系数低于指定值，就会启动复制操作。可能需要重新复制的操作有：某个 Datanode 节点失效、某个副本遭到破坏、Datanode 上的磁盘错误、或者文件的副本系数变大。  

## 10. 存储空间回收  

### 文件的删除和恢复

当用户或应用程序删除某个文件时，这个文件并没有立刻从 HDFS 中删除。实际上 HDFS 会将这个文件重命名并转移到 /trash 目录，所以只要该文件在 /trash 目录中，就可以被迅速恢复。文件在 /trash 中保存的时间通过 fs.trash.interval 配置，当超过这个时间时，Namenode 就会将文件从 namespace 中删除。删除文件会使得该文件的 block 被释放。

Namenode 在做类似的常规扫描时，Namenode 找点孤儿 block（不被任何文件包含的 block）并删除它们的元数据。然后 Datanode 在和 Namenode 交互心跳信息中，报告它所拥有的 block 子集的信息，Namenode 回复 Datanode 哪些 block 在元数据中已经不存在了，Datanode 便可以任意删除这些 block 副本了。

### 减少副本系数

同样的，当一个文件的副本系数被减小后，Namenode 会选择过剩的副本删除。其原理与上面的类似。

# 三、参考资料
- [Hadoop 分布式文件系统：架构与设计](https://hadoop.apache.org/docs/r1.0.4/cn/hdfs_design.html)
- [HDFS Architecture Guide](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html)
- [GFS 论文](http://blog.bizcloudsoft.com/wp-content/uploads/Google-File-System%E4%B8%AD%E6%96%87%E7%89%88_1.0.pdf)
- [Hadoop: The Definitive Guide](http://barbie.uta.edu/~jli/Resources/MapReduce&Hadoop/Hadoop%20The%20Definitive%20Guide.pdf)
- [Apache Hadoop HDFS Architecture](https://www.edureka.co/blog/apache-hadoop-hdfs-architecture/?utm_source=quora&utm_medium=crosspost&utm_campaign=social-media-edureka-ab)
- [HDFS 2.x High Availability Cluster Architecture](https://www.edureka.co/blog/how-to-set-up-hadoop-cluster-with-hdfs-high-availability/)
