---
layout:     post
title:      "Windows下的IDEA开发Spark SQL操作Hive"
date:       2019-05-23 02:18:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - spark

---



### 概述

其实可以用jar包提交到spark-shell 或者spark-submit上执行自己开发的Spark SQL应用程序，但是感觉提交到服务器上执行麻烦，不易于修改代码，故想在本地执行代码。

### 操作步骤

#### 启动 metastore 

~~~
[hadoop@hadoop001 bin]$ hive --service metastore 
~~~

#### 启动Spark里面的hiveserver2

~~~
[hadoop@hadoop001 sbin]$ ./start-thriftserver.sh --jars ~/software/mysql-connector-java-5.1.43-bin.jar 
~~~



#### 启动spark中的beeline

~~~
[hadoop@hadoop001 bin]$ ./beeline -u jdbc:hive2://hadoop001:10000 -n hadoop         
~~~

**注意：可以不启动 hive2 和 beeline，只启动 metastore;为了后面的测试启动了hive2和beeline**

#### 在IEDA的resources里面添加hive-site.xml

~~~
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
<configuration>
   <property>  
        <name>hive.metastore.uris</name>  
        <value>thrift://192.168.100.111:9083</value>  
   </property>
</configuration>
~~~



#### 具体的代码

~~~
package SparkSQL_UDF

import org.apache.spark.sql.functions.count
import org.apache.spark.sql.{SaveMode, SparkSession}

/**
 * Spark SQL 操作 Hive
 * Hive的元数据存储在MySQL中
 */
object HiveApp {
  def main(args: Array[String]) {
    System.setProperty("HADOOP_USER_NAME", "hadoop")

    val spark = SparkSession.builder()
      .appName("HiveApp")
      .config("spark.driver.extraJavaOptions","-XX:PermSize=128M -XX:MaxPermSize=256M")
      .config("spark.driver.memory","2g")
      .config("spark.sql.autoBroadcastJoinThreshold","-1")
      .master("local[2]")
      .enableHiveSupport()// 支持Hive，需要将hive-site.xml放到resources目录下
      .getOrCreate()

    //spark.sql("show databases").show
    //spark.sql("show tables").show

    spark.sql("use hive")
    spark.sql("select deptno,count(1) from emp group by deptno").show(false)

    val empDF = spark.table("hive.emp")
    //empDF.show(false)
    val tmp = empDF.select("deptno","sal").filter("deptno is not null").filter(empDF.col("sal")>1500)
    tmp.show(false)

    // 设置shuffle时的分区数量
    // 在生产环境中一定要注意设置spark.sql.shuffle.partitions，默认是200
    /*spark.sqlContext.setConf("spark.sql.shuffle.partitions","10")
*/
    // saveAsTable时，字段不能包含 invalid character(s) among " ,;{}()\n\t="，需要先将字段重命名
    val waitWrDF = empDF.select("deptno")
      .groupBy("deptno")
      .agg(count("deptno").as("mount"))
    waitWrDF.show()
    waitWrDF.write
      .mode(SaveMode.Overwrite)
      .saveAsTable("hive.empout")// 写入到Hive表，需要打成jar提交到集群中运行

	spark.sql("show databases").show // 查看MySQL里面的数据库信息

    spark.stop()
  }
}
~~~



### 注意事项

**注意：System.setProperty("HADOOP_USER_NAME", "hadoop")**

`如果不加上上面的代码会报权限的错误 users  with view permissions: Set(HUAWEI); groups with view permissions: Set(); users  with modify permissions: Set(HUAWEI); groups with modify permissions: Set()`



**注意：将数据书写到hive的时候，HDFS目录权限问题**

```
ERROR log: Got exception: org.apache.hadoop.security.AccessControlException Permission denied: user=HUAWEI, access=WRITE, inode="/user/hive/warehouse":hadoop:supergroup:drwxr-xr-x
```

**注意：启动hive2的时候会遇到/spark/lib/spark-assembly-*.jar: No such file or directory**

~~~
 请修改$HIVE_HOME/bin/hive 启动脚本文件,在116行
 
 // 定位到位置,上面一行是原有的,下一行是修改的
#sparkAssemblyPath=`ls ${SPARK_HOME}/lib/spark-assembly-*.jar`
sparkAssemblyPath=`ls ${SPARK_HOME}/jars/*.jar`
~~~

**修改HDFS目录的权限**

~~~
[hadoop@hadoop001 ~]$ hdfs dfs -chmod 777 /user/hive/warehouse
~~~



### 查看结果

**Hive里面查看是否存在empout**

~~~
hive (hive)> select * from empout;
OK
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/home/hadoop/app/hive-1.1.0-cdh5.7.0/lib/hive-exec-1.1.0-cdh5.7.0.jar!/shaded/parquet/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hadoop/app/hive-1.1.0-cdh5.7.0/lib/hive-jdbc-1.1.0-cdh5.7.0-standalone.jar!/shaded/parquet/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hadoop/app/hive-1.1.0-cdh5.7.0/lib/parquet-hadoop-bundle-1.5.0-cdh5.7.0.jar!/shaded/parquet/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [shaded.parquet.org.slf4j.helpers.NOPLoggerFactory]
NULL    0
20      5
10      3
30      6
Time taken: 0.296 seconds, Fetched: 4 row(s)
~~~



**在控制台中查看是否输出hive的元信息中的数据库名称**

~~~
+------------+
|databaseName|
+------------+
|     default|
|        hive|
+------------+
~~~





