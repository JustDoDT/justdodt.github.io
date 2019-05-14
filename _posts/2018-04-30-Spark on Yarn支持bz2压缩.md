---
layout:     post
title:      "Spark on Yarn支持bz2压缩"
date:       2018-04-17 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - spark
---

### 1. 修改配置文件

#### 1.1 在hdfs-core.xml文件添加如下内容

    <property>
        <name>io.compression.codecs</name>
        <value>org.apache.hadoop.io.compress.GzipCodec,
               org.apache.hadoop.io.compress.DefaultCodec,
               org.apache.hadoop.io.compress.BZip2Codec,
               org.apache.hadoop.io.compress.SnappyCodec,
               com.hadoop.compression.lzo.LzoCodec,
               com.hadoop.compression.lzo.LzopCodec
        </value>
    </property>
    
    <property>
        <name>io.compression.codec.lzo.class</name>
        <value>com.hadoop.compression.lzo.LzopCodec</value>
    </property>

#### 1.2 在spark-env.sh中添加如下内容

    export LD_LIBRARY_PATH=:/usr/local/lib
    
#### 1.3 在spark-defaults.conf添加如下内容

    spark.driver.extraClassPath        /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/share/hadoop/common/hadoop-lzo-0.4.21-SNAPSHOT.jar
    spark.executor.extraClassPath             /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/share/hadoop/common/hadoop-lzo-0.4.21-SNAPSHOT.jar
    
### 2. 启动Spark On Yarn
 
    [hadoop@hadoop001 ~]$ spark-shell --master yarn
    .
    .
    .
    Spark context Web UI available at http://hadoop001:4040
    Spark context available as 'sc' (master = yarn, app id = application_1555619527186_0003).
    Spark session available as 'spark'.
    Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_144)
    Type in expressions to have them evaluated.
    Type :help for more information.
    
    scala> sc.textFile("hdfs://hadoop001:8020/input/wordcount.txt").flatMap(x=>x.split(",")).map((_,1)).reduceByKey(_+_).saveAsTextFile("out/put")

### 3. 查看结果
 
    [hadoop@hadoop001 ~]$ hdfs dfs -ls /out/put
    Found 3 items
    -rw-r--r--   1 hadoop supergroup          0 2019-04-19 04:49 /out/put/_SUCCESS
    -rw-r--r--   1 hadoop supergroup         64 2018-04-19 04:49 /out/put/part-00000.bz2
    -rw-r--r--   1 hadoop supergroup         57 2018-04-19 04:49 /out/put/part-00001.bz2
 
    [hadoop@hadoop001 ~]$ hdfs dfs -text /out/put/part-00000.bz2
    18/04/19 08:02:09 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
    18/04/19 08:02:09 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev f1deea9a313f4017dd5323cb8bbb3732c1aaccc5]
    18/04/19 08:02:09 INFO bzip2.Bzip2Factory: Successfully loaded & initialized native-bzip2 library system-native
    18/04/19 08:02:09 INFO compress.CodecPool: Got brand-new decompressor [.bz2]
    (Hello,4)
    (World,3)
    
    [hadoop@hadoop001 ~]$ hdfs dfs -text /out/put/part-00001.bz2
    18/04/19 08:02:25 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
    18/04/19 08:02:25 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev f1deea9a313f4017dd5323cb8bbb3732c1aaccc5]
    18/04/19 08:02:25 INFO bzip2.Bzip2Factory: Successfully loaded & initialized native-bzip2 library system-native
    18/04/19 08:02:25 INFO compress.CodecPool: Got brand-new decompressor [.bz2]
    (China,2)
    (Hi,1)
    
    
    
    
    
    
    
