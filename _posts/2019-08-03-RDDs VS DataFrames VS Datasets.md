---
layout:     post
title:      "RDDs VS DataFrames VS Datasets"
date:       2019-08-03 01:28:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Spark

---



### 概述

Apache Spark对开发人员的吸引力之一是其易于使用的API，用于在大型数据集上跨语言操作：Scala，Java，Python和R。在本博客中，我将探讨Apache Spark 2.2及更高版本中提供的三组API-RDD，DataFrame和Datasets; 为什么以及何时应该使用每一套; 概述他们的表现和优化效益; 并枚举何时使用DataFrames和Datasets而不是RDD的方案。 大多数情况下，我将专注于DataFrames和Datasets，因为在Apache Spark 2.0中，这两个API是统一的。

这种统一背后的主要动机是我们通过限制您必须学习的概念数量以及提供处理结构化数据的方法来简化Spark。 通过结构，Spark可以提供更高级别的抽象和API作为特定于域的语言结构。

### Resilient Distributed Dataset (RDD)

RDD自成立以来就是Spark中面向用户的主要API。 在Spark Core里面，RDD是数据元素的不可变分布式集合，跨集群中的节点进行分区，可以与提供转换和操作的低级API并行运行。

#### 什么时候用 RDDs?

>**在以下情况下考虑使用RDD的这些方案或常见用例：**
>
>- 您希望对数据集进行低级转换以及操作和控制;
>- 您的数据是非结构化的，例如媒体流或文本流;
>- 您希望使用函数式编程构造来操作数据而不是域特定的表达式;
>- 在按名称或列处理或访问数据属性时，您不关心强制执行模式（如列式格式）; 
>- 对于结构化和半结构化数据，您可以放弃 DataFrames 和 Datasets 提供的一些优化和性能优势。



#### Apache Spark2.0 中 RDD发生了什么？

>  您可能会问：RDD是否会被降级为二等公民？ 他们被弃用了吗？
>
> 答案是响亮的NO！
>
> 更重要的是，正如您将在下面注意到的，您可以通过简单的API方法调用在DataFrame或Dataset和RDD之间无缝移动 - 并且DataFrame和数据集构建在RDD之上。



### DataFrames

> - 与RDD一样，DataFrame是不可变的分布式数据集合。 与RDD不同，数据被组织到命名列中，就像关系数据库中的表一样。 DataFrame旨在使大型数据集处理变得更加容易，它允许开发人员将结构强加到分布式数据集合上，从而实现更高级别的抽象; 它提供了一个特定于域的语言API来处理您的分布式数据; 除了专业的数据工程师之外，还可以让更多的受众访问Spark。
>
> - 在我们的Apache Spark 2.0网络研讨会和后续博客的预览中，我们提到在Spark 2.0中，DataFrame API将与Datasets API合并，统一跨库的数据处理功能。 由于这种统一，开发人员现在学习或记忆的概念较少，并且使用一个名为Dataset的高级且类型安全的API。



![spark](/img/Spark/SparkSQL/spark_rdd_df_ds1.png)





### Datasets

> - 从Spark 2.0开始，Dataset具有两个不同的API特征：**强类型API** 和 **无类型API**，如下表所示。 从概念上讲，将 DataFrame 视为通用对象` Dataset [Row]` 的集合的别名，其中 Row 是通用的无类型 JVM 对象。 相比之下，数据集是强类型JVM对象的集合，由您在Scala中定义的案例类或Java中的类决定。



**Typed and Un-typed APIs**

| Language | Main Abstraction                                |
| -------- | ----------------------------------------------- |
| Scala    | Dataset[T] & DataFrame (alias for Dataset[Row]) |
| Java     | Dataset[T]                                      |
| Python*  | DataFrame                                       |
| R*       | DataFrame                                       |

> **注意：**
>
> 由于Python和R没有编译时类型安全性，我们只有非类型化的API，即DataFrames。



### Benefits of Dataset APIs

`作为Spark开发人员，您可以通过多种方式使用Spark 2.0中的DataFrame和Dataset统一API。`

> #### 1.静态类型和运行时类型安全
>
> 将静态类型和运行时安全性视为一种频谱，SQL对数据集的限制最小。例如，在Spark SQL字符串查询中，在运行时（可能代价很高）之前您不会知道语法错误，而在DataFrames和Datasets中，您可以在编译时捕获错误（这可以节省开发人员的时间和成本）。也就是说，如果在DataFrame中调用不属于API的函数，编译器将捕获它。但是，它不会在运行时检测到不存在的列名。
>
> 在频谱的最远端是 Dataset，限制性最强。由于Dataset API都表示为lambda函数和JVM类型对象，因此在编译时将检测到类型参数的任何不匹配。此外，使用 Dataset 时，也可以在编译时检测分析错误，从而节省开发人员的时间和成本。
>
> 所有这些都转化为Spark代码中语法和分析错误的类型安全谱，Dataset 对于开发人员来说是最具限制性的，但却很有效。

![spark](/img/Spark/SparkSQL/spark_rdd_df_ds2.png)



>#### 2.结构化和半结构化数据的高级抽象和自定义视图
>
>DataFrames 作为 Datasets[Row]的集合，将结构化自定义视图呈现为半结构化数据。 例如，假设您有一个巨大的IoT设备事件数据集，表示为JSON。 由于JSON是一种半结构化格式，因此它非常适合使用数据集作为强类型特定 Dataset[DeviceIoTData]的集合。
>
>~~~
>{"device_id": 198164, "device_name": "sensor-pad-198164owomcJZ", "ip": "80.55.20.25", "cca2": "PL", "cca3": "POL", "cn": "Poland", "latitude": 53.080000, "longitude": 18.620000, "scale": "Celsius", "temp": 21, "humidity": 65, "battery_level": 8, "c02_level": 1408, "lcd": "red", "timestamp" :1458081226051}
>~~~



**您可以将每个JSON条目表达为DeviceIoTData，一个自定义对象，带有Scala案例类。**

~~~
case class DeviceIoTData (battery_level: Long, c02_level: Long, cca2: String, cca3: String, cn: String, device_id: Long, device_name: String, humidity: Long, ip: String, latitude: Double, lcd: String, longitude: Double, scale:String, temp: Long, timestamp: Long)
~~~



**Next, we can read the data from a JSON file.**

~~~
// read the json file and create the dataset from the 
// case class DeviceIoTData
// ds is now a collection of JVM Scala objects DeviceIoTData
val ds = spark.read.json(“/databricks-public-datasets/data/iot/iot_devices.json”).as[DeviceIoTData]
~~~



**在以上的代码发生了3件事**

- Spark读取JSON，推断架构，并创建DataFrame集合。
- 此时，Spark将您的数据转换为 DataFrame = Dataset [Row]，这是一个通用Row对象的集合，因为它不知道确切的类型。
- 现在，Spark根据类DeviceIoTData的指示转换数据集[Row]  - > Dataset [DeviceIoTData]类型特定的Scala JVM对象。

> 我们大多数人都使用结构化数据，习惯于以列式方式查看和处理数据或访问对象中的特定属性。 使用Dataset作为Dataset [ElementType]类型对象的集合，您可以无缝地获得强类型JVM对象的编译时安全性和自定义视图。 从上面的代码中生成的强类型数据集[T]可以使用高级方法轻松显示或处理。



> #### 3. 具有结构的API的易用性
>
> 尽管结构可能会限制Spark程序可以对数据执行的操作，但它引入了丰富的语义和一组简单的特定于域的操作，这些操作可以表示为高级构造。 但是，大多数计算都可以使用Dataset的高级API来完成。 例如，通过访问数据集类型对象的DeviceIoTData而不是使用RDD行的数据字段来执行agg，select，sum，avg，map，filter或groupBy操作要简单得多。
>
> 在特定于域的API中表达计算比使用关系代数类型表达式（在RDD中）更简单，更容易。 例如，下面的代码将filter（）和map（）创建另一个不可变的数据集。



~~~
// Use filter(), map(), groupBy() country, and compute avg() 
// for temperatures and humidity. This operation results in 
// another immutable Dataset. The query is simpler to read, 
// and expressive

val dsAvgTmp = ds.filter(d => {d.temp > 25}).map(d => (d.temp, d.humidity, d.cca3)).groupBy($"_3").avg()

//display the resulting dataset
display(dsAvgTmp)
~~~



> #### 4. 性能和优化
>
> 除了上述所有优点外，您还不能忽视使用DataFrames和Dataset API时的空间效率和性能提升，原因有两个。
>
> - `首先，`因为DataFrame和Dataset API构建在Spark SQL引擎之上，所以它使用Catalyst生成优化的逻辑和物理查询计划。 在R，Java，Scala或Python DataFrame / Dataset API中，所有关系类型查询都经历相同的代码优化器，从而提供空间和速度效率。 Dataset[T] 类型API针对数据工程任务进行了优化，而无类型 Dataset[ROW]（DataFrame的别名）甚至更快，适用于交互式分析。



![spark](/img/Spark/SparkSQL/spark_rdd_df_ds3.png)





> - `其次，`由于Spark作为编译器理解您的数据集类型 JVM 对象，它使用编码器将特定于类型的 JVM 对象映射到 Tungsten 的内部存储器表示。 因此，Tungsten Encoders 可以有效地`序列化/反序列化` JVM 对象，并生成可以以超高速度执行的紧凑字节码。



### 我什么时候应该使用DataFrames或Datasets？

- 如果您需要丰富的语义，高级抽象和特定于域的API，请使用DataFrame或Dataset。
- 如果您的处理需要高级表达式，过滤器，映射，聚合，平均值，求和，SQL查询，列式访问以及对半结构化数据使用lambda函数，请使用DataFrame或Dataset。
- 如果您希望在编译时具有更高的类型安全性，需要类型化的 JVM 对象，利用 Catalyst 优化，并从Tungsten的高效代码生成中受益，请使用数据集。
- 如果要跨 Spark 库统一和简化API，请使用DataFrame或Dataset。
- 如果您是R用户，请使用DataFrames。
- 如果您是Python用户，请使用DataFrames并在需要更多控制时使用RDD。

**请注意，通过简单的方法调用.rdd，您始终可以无缝地互操作或从DataFrame和/或数据集转换为`.RDD`。 例如，**

~~~
// select specific fields from the Dataset, apply a predicate
// using the where() method, convert to an RDD, and show first 10
// RDD rows
val deviceEventsDS = ds.select($"device_name", $"cca3", $"c02_level").where($"c02_level" > 1300)
// convert to RDDs and take the first 10 rows
val eventsRDD = deviceEventsDS.rdd.take(10)
~~~



### 总结

总之，选择何时使用RDD或DataFrame和/或数据集似乎是显而易见的。 前者为您提供低级功能和控制，后者允许自定义视图和结构，提供高级和特定于域的操作，节省空间，并以超高速度执行。

当我们研究从Spark的早期版本中学到的经验教训 - 如何为开发人员简化Spark，如何优化并使其具有高性能 - 我们决定将低级 RDD API 提升为高级抽象，如DataFrame和Dataset以及 在Catalyst优化器和Tungsten之上的库之间构建统一的数据抽象。

选择满足您需求和用例的 one-DataFrames 和/或 Dataset 或 RDDs API，但如果您陷入使用结构和半结构化数据的大多数开发人员的阵营中，我不会感到惊讶。



### 参考资料

本文是翻译的Spark 砖厂的博客，下面是博客原文。

- [A Tale of Three Apache Spark APIs: RDDs vs DataFrames and Datasets](<https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html>)

