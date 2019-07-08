---
layout:     post
title:      "浅析Kafka的failover"
date:       2019-06-17 21:03:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Kafka
---



### 概述

这里讨论 kafka 的 failover 的前提是在0.8版本后， kafka 提供了 replica 机制。 
对于0.7版本不存在 failover 的说法，因为任意一个 broker dead 都会导致上面的数据不可读，从而导致服务中断。

下面简单的介绍一下 0.8中加入的 replica 机制和相应的组件。

#### Replica机制

基本思想大同小异，如下图 (Ref.2):

![Kafka](/img/Kafka/failover1.png) 



图中有4个 kafka brokers，并且Topic1有四个 partition（用蓝色表示）分布在4个 brokers 上，为 leader replica； 
且每个 partition 都有两个 follower replicas（用橘色表示），分布在和 leader replica 不同的 brokers。 
这个分配算法很简单，有兴趣的可以参考kafka的design。

#### Replica组件

为了支持replica机制，主要增加的两个组件是，Replica Manager和Controller， 如下图：

![Kafka](/img/Kafka/failover2.png) 



#### Replica Manager

每个broker server都会创建一个Replica Manager，所有的数据读写都需要经过它，0.7版本，kafka会直接从LogManager中读数据，但是在增加replica机制后，只有leader replica 可以响应数据的读写请求。所以，Replica Manager 需要管理所有 partition 的 replica 状态，并响应读写请求，以及其他和 replica 相关的操作。

#### Controller

大家可以看到，每个 partition 都有一个 leader replica，和若干的follower replica，那么谁来决定谁是leader？

你说你有 zookeeper，但是用zk为每个 partition 做选举，效率太低，而且zk会不堪重负。

所以现在的通用做法是，只用zk选一个master节点，然后由这个master 节点来做其他的所有仲裁工作。

kafka 的做法就是在brokers 中选出一个作为 controller, 来做为 master 节点，从而仲裁所有的 partition 的 leader 选举。

下面我们会从如下几个方面来解释 failover 机制， 
先从 client 的角度看看当 kafka 发生 failover 时，数据一致性问题。 
然后从 Kafka 的各个重要组件，Zookeeper，Broker， Controller 发生 failover 会造成什么样的影响？ 



#### 从Client 的角度

#### 从producer的角度，发的数据是否会丢？

除了要打开 replica 机制，还取决于 produce 的 request.required.acks 的设置，

- acks = 0，发就发了，不需要 ack，无论成功与否 ；
- acks = 1，当写 leader replica 成功后就返回，其他的 replica 都是通过fetcher去异步更新的，当然这样会有数据丢失的风险，如果leader的数据没有来得及同步，leader挂了，那么会丢失数据；
- acks = –1(all), 要等待所有的replicas都成功后，才能返回；这种纯同步写的延迟会比较高。

所以，一般的情况下，设成1，在极端情况下，是有可能丢失数据的； 
如果可以接受较长的写延迟，可以选择将 acks 设为 –1。

#### 从consumer 的角度, 是否会读到不一致的数据？

首先无论是 high-level 或 low-level consumer，我们要知道他是怎么从 kafka 读数据的？

![Kafka](/img/Kafka/failover3.png) 

kafka 的 log patition 存在文件中，并以 offset 作为索引，所以 consumer 需要对于每个 partition 记录上次读到的 offset （high-level和low-level的区别在于是 kafka 帮你记，还是你自己记）；

所以如果 consumer dead，重启后只需要继续从上次的 offset 开始读，那就不会有不一致的问题。

但如果是 Kafka broker dead，并发生 partition leader 切换，如何保证在新的 leader 上这个 offset 仍然有效？ 
Kafka 用一种机制，即 committed offset，来保证这种一致性，如下图(Ref.2)

![Kafka](/img/Kafka/failover4.png) 

log 除了有 log end offset 来表示 log 的末端，还有一个 committed offset， 表示有效的 offset； 
committed offset 只有在**所有 replica 都同步完该 offset** 后，才会被置为该offset； 
所以图中 committed 置为2， 因为 broker3 上的 replica 还没有完成 offset 3 的同步； 
所以这时，offset 3 的 message 对 consumer 是不可见的，consumer最多只能读到 offset 2。 
如果此时，leader dead，无论哪个 follower 重新选举成 leader，都不会影响数据的一致性，因为consumer可见的offset最多为2，而这个offset在所有的replica上都是一致的。

所以在一般正常情况下，当 kafka 发生 failover 的时候，consumer 是不会读到不一致数据的。特例的情况就是，当前 leader 是唯一有效的 replica，其他replica都处在完全不同步状态，这样发生 leader 切换，一定是会丢数据的，并会发生 offset 不一致。

### Zookeeper Failover

Kafka 首先对于 zookeeper 是强依赖，所以 zookeeper 发生异常时，会对数据造成如何的影响？

#### Zookeeper Dead

如果 zookeeper dead，broker 是无法启动的，会报ZK异常，连接拒绝。这种异常，有可能是 zookeeper dead，也有可能是网络不通，总之就是连不上 zookeeper。 这种 case，kafka完全不工作，直到可以连上 zookeeper 为止。

#### Zookeeper Hang

其实上面这种情况比较简单，比较麻烦的是 zookeeper hang，可以说 kafka 的80%以上问题都是由于这个原因 
zookeeper hang 的原因有很多，主要是 zk 负载过重，zk 所在主机 cpu，memeory 或网络资源不够等

zookeeper hang 带来的主要问题就是 session timeout，这样会触发如下的问题，

a. Controller Fail，Controller 发生重新选举和切换，具体过程参考下文。

b. Broker Fail，导致partition的leader发生切换或partition offline，具体过程参考下文。

c. Broker 被 hang 住 。 
这是一种比较特殊的 case，出现时在 server.log 会出现如下的log，

~~~
server.log： “INFO I wrote this conflicted ephemeral node [{"jmx_port":9999,"timestamp":"1444709  63049","host":"10.151.4.136","version":1,"port":9092}] at /brokers/ids/1 a while back in a different session, hence I will backoff for this node to be deleted by Zookeeper and retry (kafka.utils.ZkUtils$)”
~~~

即 **zk 的 session 过期和 ephemeral node 删除并不是一个原子操作;** 
出现的case如下：

- 在极端case下，zk 触发了 session timeout，但还没来得及完成 /brokers/ids/1 节点的删除，就被 hang 住了，比如是去做很耗时的 fsync 操作 。
- 但是 broker 1 收到 session timeout 事件后，会尝试重新去 zk 上创建 /brokers/ids/1 节点，可这时旧的节点仍然存在，所以会得到 NodeExists，其实这个是不合理的，因为既然 session timeout，这个节点就应该不存在。
- 通常的做法，既然已经存在，我就不管了，该干啥干啥去；问题是一会 zk 从 fsync hang 中恢复了，他会记得还有一个节点没有删除，这时会去把 /brokers/ids/1 节点删除。
- 结果就是对于client，虽然没有再次收到 session 过期的事件，但是 /brokers/ids/1 节点却不存在了。

所以这里做的处理是，在前面发现 NodeExists 时，while true 等待，一直等到 zk 从 hang 中恢复删除该节点，然后创建新节点成功，才算完； 
这样做的结果是这个broker也会被一直卡在这儿，等待该节点被成功创建。

### Broker Failover

Broker 的 Failover，可以分为两个过程，一个是 broker failure， 一个是 broker startup。

#### 新加 broker

在谈failover之前，我们先看一个更简单的过程，就是新加一个全新的 broker： 
首先明确，`新加的 broker 对现存所有的 topic 和 partition，不会有任何影响`； 
因为一个 topic 的 partition 的所有 replica 的 assignment 情况，在创建时就决定了，并不会自动发生变化，除非你手动的去做 reassignment。 
所以新加一个 broker，所需要做的只是大家同步一下元数据，大家都知道来了一个新的 broker，当你创建新的 topic 或 partition 的时候，它会被用上。

 

#### Broker Failure

首先明确，这里的 broker failure，并不一定是 broker server 真正的 dead了， 只是指该 broker 所对应的 zk ephemeral node ，比如/brokers/ids/1，发生 session timeout； 
当然发生这个的原因，除了server dead，还有很多，比如网络不通；但是我们不关心，只要出现 sessioin timeout，我们就认为这个 broker 不工作了； 
会出现如下的log，

~~~
controller.log： “INFO [BrokerChangeListener on Controller 1]: Newly added brokers: 3, deleted brokers: 4, all live brokers: 3,2,1 (kafka.controller.ReplicaStateMachine$BrokerChangeListener)” “INFO [Controller 1]: Broker failure callback for 4 (kafka.controller.KafkaController)”
~~~



当一个 broker failure 会影响什么，其实对于多 replicas 场景，一般对最终客户没啥影响。 
`只会影响哪些 leader replica 在该 broker 的 partitions； 需要重新做 leader election，如果无法选出一个新的 leader，会导致 partition offline。`
因为如果只是 follow replica failure，不会影响 partition 的状态，还是可以服务的，只是可用 replica 少了一个；需要注意的是，kafka 是不会自动补齐失败的replica的，即坏一个少一个； 
但是对于 leader replica failure，就需要重新再 elect leader，前面已经讨论过，新选取出的 leader 是可以保证 offset 一致性的；

`Note：` 

其实这里的一致性是有前提的，即除了 fail 的 leader，在 ISR（in-sync replicas） 里面还存在其他的 replica；顾名思义，ISR，就是能 catch up with leader 的 replica。 虽然 partition 在创建的时候，会分配一个 AR（assigned replicas），但是在运行的过程中，可能会有一些 replica 由于各种原因无法跟上 leader，这样的 replica 会被从 ISR 中去除。 所以 ISR <= AR； 如果，ISR 中 没有其他的 replica，并且允许 unclean election，那么可以从 AR 中选取一个 leader，但这样一定是丢数据的，无法保证 offset 的一致性。

#### Broker Startup

这里的 startup，就是指 failover 中的 startup，会出现如下的log，

~~~
controller.log： “INFO [BrokerChangeListener on Controller 1]: Newly added brokers: 3, deleted brokers: 4, all live brokers: 3,2,1 (kafka.controller.ReplicaStateMachine$BrokerChangeListener)” “INFO [Controller 1]: New broker startup callback for* *3 (kafka.controller.KafkaController)”
~~~



过程也不复杂，先将该 broker 上的所有的 replica 设为 online，然后触发 offline partition 或 new partition 的 state 转变为 online； 
所以 broker startup，`只会影响 offline partition 或 new partition，让他们有可能成为 online`。 
那么对于普通的已经 online partition，影响只是多一个可用的 replica，那还是在它完成catch up，被加入 ISR 后的事。

`Note: `

Partition 的 leader 在 broker failover 后，不会马上自动切换回来，这样会产生的问题是，broker间负载不均衡，因为所有的读写都需要通过 leader。 为了解决这个问题，在server的配置中有个配置，auto.leader.rebalance.enable，将其设为true； 这样 Controller 会启动一个 scheduler 线程，定期去为每个 broker 做 rebalance，即发现如果该 broker 上的 imbalance ratio 达到一定比例，就会将其中的某些 partition 的 leader，进行重新 elect 到原先的 broker 上。

### Controller Failover

前面说明过，某个 broker server 会被选出作为 Controller，这个选举的过程就是依赖于 zookeeper 的 ephemeral node，谁可以先在**"/controller"**目录创建节点，谁就是 controller； 
所以反之，我们也是 watch 这个目录来判断 Controller 是否发生 failover 或 变化。Controller 发生 failover 时，会出现如下 log：

~~~
controller.log： “INFO [SessionExpirationListener on 1], ZK expired; shut down all controller components and try to re-elect (kafka.controller.KafkaController$SessionExpirationListener)”
~~~



Controller 主要是作为 master 来仲裁 partition 的 leader 的，并维护 partition 和 replicas 的状态机，以及相应的 zk 的 watcher 注册；

Controller 的 failover 过程如下：

- 试图去在“/controller” 目录抢占创建 ephemeral node；
- 如果已经有其他的 broker 先创建成功，那么说明新的 controller 已经诞生，更新当前的元数据即可；
- 如果自己创建成功，说明我已经成为新的 controller，下面就要开始做初始化工作，
- 初始化主要就是创建和初始化 partition 和 replicas 的状态机，并对 partitions 和 brokers 的目录的变化设置 watcher。

可以看到，单纯 Controller 发生 failover，是不会影响正常数据读写的，只是 partition 的 leader 无法被重新选举，如果此时有 partition 的 leader fail，会导致 partition offline； 
但是 Controller 的 dead，往往是伴随着 broker 的 dead，所以在 Controller 发生 failover 的过程中，往往会出现 partition offline， 导致数据暂时不可用。





### 参考资料

- [Apche Kafka 的生与死 – failover 机制详解](<https://www.cnblogs.com/fxjwind/p/4972244.html>)








