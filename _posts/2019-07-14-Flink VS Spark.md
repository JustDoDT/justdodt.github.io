---
layout:     post
title:      "Flink VS Spark"
date:       2019-07-14 12:27:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Flink
---



### Flink

定义：基于数据流的有状态计算

### 对比Spark

#### 1.定位

- Spark：流是批的特例（Spark）
- Flink: 批是流的特例（Flink）

#### 2.数据模型

- Spark:RDD集合，依靠 lineage 做恢复，做容错，存在宽窄依赖
- Flink: 数据量和 event 的序列，依靠 checkpoint 做恢复，保证一致性

- DAG：
  - 来一条数据处理一条，传输到下一个节点，也可以优化批次处理，其中 checkpoint 可以不用落盘，这样可以提高性能以及低延迟

#### 3.处理模型

- Spark
  - 支持各种数据处理，对机器学习迭代训练支持比较好（基于RDD本身模型）
- Flink
  - 有状态的计算，尤其API层级多样，支持丰富

#### 4. 抽象层次

- Spark
  - 一套API
- Flink
  - DataSet 和 DataStream 是独立的 API

#### 5. 内存管理

- Spark
  - JVM ----> Tungsten
- Flink 
  - 自己管理内存

#### 6. 支持语言

- Spark 比 Flink 支持的丰富，支持java,scala,python,R
- flink 目前只支持 java , scala

#### 7. SQL支持

- Spark 比 Flink 支持的丰富

#### 8. 外部数据源

- Spark: NoSQL ，ORC，Parquet , RDBMS
- Flink: Map / Reduce , InputFormat

### Flink适用场景

#### 1. Event-driven Applications

- 定义：
  - stateful app
  - checkpoint 保证容错，拉取 remote data 做持久化存储

- 优势：
  - 读取本地数据而非远程数据库
  - 分层体系多个应用可以共享一个数据库
- 如何支持：
  - 有一些语言支持 time , window , processfunction 
  - savepoints: 一个一致的映像文件
- 案例：
  - XX检测，web应用（社交网络）

#### 2.Data Analytics Applications

- 定义：
  - 无界处理
  - 依赖内部 state 做 flink 故障恢复机制
- 优势：
  - 降低延迟
- 如何支持：
  - anis 的 SQL 接口，对于批处理和流处理提供了统一的语义
- 案例：
  - 网络质量监控，大规模图分析

#### 3. Data Pipeline Applications

- 定义：
  - 类似于 ETL ，但并非周期性操作，而是连续性操作，这会降低目标间数据移动的延迟
- 优势：
  - 降低延迟，更加通用
- 如何支持：
  - 基于多层 API 结构以及多种多样的 Connectos
- 案例：
  - 电商的实时搜索的索引构建，电商的连续的 ETL

