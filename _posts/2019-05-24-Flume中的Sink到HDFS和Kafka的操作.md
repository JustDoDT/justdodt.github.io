---
layout:     post
title:      "Flume中的Sink到HDFS和Kafka的操作"
date:       2019-05-24 02:41:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Flume

---



### 概述

Flume用来采集日志，也可以用来采集数据库中的数据，也可以修改源代码自定义自己想要的source,channel,sink类型。



### 安装

#### 安装Flume

~~~
[hadoop@hadoop001 software]$ tar -zxvf flume-ng-1.6.0-cdh5.7.0.tar.gz -C ~/app/

# $FLUME_HOME/conf/flume-env.sh 
export JAVA_HOME=/usr/java/jdk1.8.0_144
~~~



#### 安装Zookeeper

由于本人没有用Kafka里面的Zookeeper,自己安装了一个Zookeeper，其实可以用Kafka里面的Zookeeper。

~~~
[hadoop@hadoop001 software]$ tar -zxvf zookeeper-3.4.5-cdh5.7.0-src.tar.gz -C ~/app/

# 在$ZOOKEEPER_HOME/conf 里面

dataDir=/home/hadoop/app/zookeeper-3.4.5-cdh5.7.0/tmp
clientPort=2181
server.1=hadoop001:2888:3888
~~~



#### 安装Kafka

~~~
[hadoop@hadoop001 software]$ tar -zxvf kafka_2.11-0.10.0.0.tgz -C ~/app/

# 在$KAFKA_HOME/config/server.properties 

broker.id=0
port=9092
host.name=hadoop001
log.dirs=/home/hadoop/app/kafka_2.11-0.10.0.0/logs
zookeeper.connect=hadoop001:2181/kafka  

~~~

**注意：zookeeper.connect=hadoop001:2181/kafka ,一定要加上kafka不是后面会出现数据不能到kafak里面**



#### 配置环境变量

~~~
[hadoop@hadoop001 config]$ vi ~/.bash_profile 

export FLUME_HOME=/home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin
export PATH=${FLUME_HOME}/bin:$PATH

export KAFKA_HOME=/home/hadoop/app/kafka_2.11-0.10.0.0
export PATH=${KAFKA_HOME}/bin:$PATH

export ZOOKEEPER_HOME=/home/hadoop/app/zookeeper-3.4.5-cdh5.7.0
export PATH=${ZOOKEEPER_HOME}/bin:$PATH

# 生效环境变量
[hadoop@hadoop001 config]$ source ~/.bash_profile 
~~~



### 修改Flume的配置文件

~~~
[hadoop@hadoop001 conf]$ vi flume-hdfs-kafka.conf

flume-hdfs-kafka.sources = r1
flume-hdfs-kafka.channels = hdfs-channel kafka-channel
flume-hdfs-kafka.sinks = hdfs-sink kafka-sink

#配置source
flume-hdfs-kafka.sources.r1.type = TAILDIR
flume-hdfs-kafka.sources.r1.channels = hdfs-channel kafka-channel
flume-hdfs-kafka.sources.r1.filegroups = f1
flume-hdfs-kafka.sources.r1.filegroups.f1 = /home/hadoop/data/flume/data/.*
flume-hdfs-kafka.sources.r1.positionFile = /home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/logs/taildir_position.json


 #配置channel
flume-hdfs-kafka.sources.r1.selector.type = replicating
flume-hdfs-kafka.channels.hdfs-channel.type=file
flume-hdfs-kafka.channels.hdfs-channel.checkpointDir=/home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/checkpoint/hdfs
flume-hdfs-kafka.channels.hdfs-channel.dataDirs=/home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/data/hdfs


flume-hdfs-kafka.channels.kafka-channel.type=file
flume-hdfs-kafka.channels.kafka-channel.checkpointDir=/home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/checkpoint/kafka
flume-hdfs-kafka.channels.kafka-channel.dataDirs=/home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/data/kafka


#配置hdfs sink
flume-hdfs-kafka.sinks.hdfs-sink.type = hdfs
flume-hdfs-kafka.sinks.hdfs-sink.channel = hdfs-channel
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.path = hdfs://hadoop001:8020/flume/events/%y%m%d%H%M
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.filePrefix = events-
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.rollSize = 100000000
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.rollInterval = 30
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.rollCount = 0
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.round = true
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.roundValue = 10
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.roundUnit = minute
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.useLocalTimeStamp = true
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.fileType= CompressedStream
flume-hdfs-kafka.sinks.hdfs-sink.hdfs.codeC= bzip2

#配置kafka sink
flume-hdfs-kafka.sinks.kafka-sink.channel = kafka-channel
flume-hdfs-kafka.sinks.kafka-sink.type = org.apache.flume.sink.kafka.KafkaSink
flume-hdfs-kafka.sinks.kafka-sink.topic = flume-sink
flume-hdfs-kafka.sinks.kafka-sink.brokerList = hadoop001:9092
flume-hdfs-kafka.sinks.kafka-sink.requiredAcks = 1
flume-hdfs-kafka.sinks.kafka-sink.batchSize = 20
#flume-hdfs-kafka.sinks.kafka-sink.flumeBatchSize = 20
#flume-hdfs-kafka.sinks.kafka-sink.producer.acks = 1
#flume-hdfs-kafka.sinks.kafka-sink.producer.linger.ms = 1
#flume-hdfs-kafka.sinks.kafka-sink.producer.compression.type = snappy

~~~



### 启动各个组件

**首先启动ZK**

~~~
[hadoop@hadoop001 conf]$ zkServer.sh start
~~~



**再启动Kafka,并创建kafka主题**

~~~
[hadoop@hadoop001 conf]$ kafka-server-start.sh /home/hadoop/app/kafka_2.11-0.10.0.0/config/server.properties &
 
[hadoop@hadoop001 conf]$ kafka-topics.sh --create --zookeeper hadoop001:2181/kafka --replication-factor 1 --partitions 1 --topic flume_sink
~~~



**启动kafka的consumer**

~~~
[hadoop@hadoop001 conf]$ kafka-console-consumer.sh --zookeeper hadoop001:2181/kafka --topic flume-sink
~~~



**启动Flume**

~~~
[hadoop@hadoop001 conf]$ flume-ng agent --name flume-hdfs-kafka --conf /home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/conf/ --conf-file /home/hadoop/app/apache-flume-1.6.0-cdh5.7.0-bin/conf/flume-hdfs-kafka.conf -Dflume.root.logger=INFO,console
~~~



### 验证

**向配置的目录里面拷贝文件**

~~~
[hadoop@hadoop001 data]$ pwd
/home/hadoop/data/flume/data
[hadoop@hadoop001 data]$ cp /home/hadoop/data/baidu.log .
~~~



**在HDFS上查看**

~~~
[hadoop@hadoop001 conf]$ hdfs dfs -lsr /flume/events
lsr: DEPRECATED: Please use 'ls -R' instead.
drwxr-xr-x   - hadoop supergroup          0 2019-05-26 10:14 /flume/events/1905261010
-rw-r--r--   1 hadoop supergroup       5990 2019-05-26 10:14 /flume/events/1905261010/events-.1558836846289.bz2

~~~



**在kafka的consumer里面查看**

~~~
.
.
.
go2yd.com/user_upload/882314516209P09Y03X0x5onE57w9S41AMC9      103237794
baidu   CN      E       [06/Apr/2017:12:47:18 + 0800]   171.10.161.0    v2.go2yd.com    http://v1.go2yd.com/user_upload/Vc25DG6vdY57180z6s330o3YfFYQ836F7
[2019-05-26 10:42:23,984] INFO [Group Metadata Manager on Broker 0]: Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.GroupMetadataManager)
[2019-05-26 10:52:23,984] INFO [Group Metadata Manager on Broker 0]: Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.GroupMetadataManager)
[2019-05-26 11:02:23,984] INFO [Group Metadata Manager on Broker 0]: Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.GroupMetadataManager)
[2019-05-26 11:12:23,985] INFO [Group Metadata Manager on Broker 0]: Removed 0 expired offsets in 1 milliseconds. (kafka.coordinator.GroupMetadataManager)
[2019-05-26 11:22:23,984] INFO [Group Metadata Manager on Broker 0]: Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.GroupMetadataManager)
~~~



### 参考文章

- [Apache Flume 1.6.0 documentation](http://archive.cloudera.com/cdh5/cdh/5/flume-ng-1.6.0-cdh5.7.0/FlumeUserGuide.html)

- [Apache Kafka](http://kafka.apache.org/documentation/#quickstart)
