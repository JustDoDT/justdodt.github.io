---
layout:     post
title:      "HDFS出现Unable to load native-hadoop library for your platform... using builtin-java classes where applicable"
date:       2019-03-19 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
#catalog: true
tags:
    - hadoop, hdfs
---

**错误信息如下：**

    [hadoop@hadoop001 hadoop]$ hdfs  dfs -lsr /d6_hive
    lsr: DEPRECATED: Please use 'ls -R' instead.
    19/03/19 07:46:21 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    drwxr-xr-x   - hadoop supergroup          0 2019-03-19 07:15 /d6_hive/test
    [hadoop@hadoop001 hadoop]$ 



在谷歌上找了一些解决办法：

**不能解决的方法：**

执行：$ export HADOOP_ROOT_LOGGER=DEBUG,console

再执行：hdfs dfs -ls /  会打印很多日志信息，注意其中的一条日志。

Failed to load native-hadoop with error: java.lang.UnsatisfiedLinkError，这表明是java.library.path出了问题，

解决方案是在文件hadoop-env.sh中增加：

export HADOOP_OPTS="-Djava.library.path=${HADOOP_HOME}/lib/native"  

但是按照上述操作了，发现还是没有解决此问题。



**可以解决此问题的方法：**

方法一：

在 /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/etc/hadoop/log4j.properties  中添加

    log4j.logger.org.apache.hadoop.util.NativeCodeLoader=ERROR
    
然后重启hdfs

    [hadoop@hadoop001 hadoop]$ 
    [hadoop@hadoop001 hadoop]$ hdfs dfs -lsr /d6_hive/test
    lsr: DEPRECATED: Please use 'ls -R' instead.
    [hadoop@hadoop001 hadoop]$ 

参考：https://www.cnblogs.com/kevinq/p/5103653.html

