---
layout:     post
title:      "浅析Apache Zeppelin"
date:       2020-02-08 17:04:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Zeppelin


---

## 概述

`2013`年，`ZEPL`（以前称为`NFLabs`，是一家韩国的数据分析公司）启动了`Zeppelin`项目。

`2014`年`12`月`23`日，`Zeppelin`项目成为`Apache Software Foundation`中的孵化项目。

`2016`年`6`月`18`日，`Zeppelin`项目从`Apache`的孵化项目毕业，成为`Apache Software Foundation`的顶级项目。

`2020`年`1`月`19`日，`Apache Zepplin`已经到了`0.8.2GA`，`0.9.0-SNAPSHOT`。

## 什么是Apache Zeppelin

`Apache Zeppelin`是一个基于`Web`的交互式数据分析开源框架，提供了数据分析、数据可视化等功能，类似于`Jupyter Notebook`，不过功能比`Jupyter Notebook`强大很多，她是属于企业级产品，支持超过`20`种后端语言，比如`Java/Scala/Python/SQL/MarkDown/Shell`等。开发者可以自定义开发更多的解释器为`Zeppelin`添加执行引擎，官方现在支持的执行引擎有`Spark/JDBC/Python/Alluxio/Beam/Cassandra/Elasticsearch/Flink/HBase/HDFS/Hive/Kylin/Livy/MarkDown/Neo4j/Pig/PostgreSQL/R`等。

### Apache Zeppelin 架构简介

`Apache Zeppelin`的架构比较简单直观（如下图所示），总共分为`3`层

- `Zeppelin` 前端
- `Zeppelin Server`
- `Zeppelin Interpreter`

![1581148506356](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581148506356.png)
 ` Zeppelin`前端是基于`AngularJS`。
 `Zeppelin Server`是一个基于`Jetty`的轻量级`Web Server`，主要负责以下一些功能：

- 登陆权限管理
- `Zeppelin`配置信息管理
- `Interpreter` 配置信息和生命周期管理
- `Note`存储管理
- 插件机制管理

`Zeppelin Interpreter`组件是指各个计算引擎在`Zeppelin`这边的适配。比如`Python`，`Spark`，`Flink`等等。每个`Interpreter`都运行在一个`JVM`进程里，这个`JVM`进程可以是和`Zeppelin Server`在同一台机器上（默认方式），也可以运行在`Zeppelin`集群里的其他任何机器上或者`K8s`集群的一个`Pod`里，这个由`Zeppelin`的不同`InterpreterLauncher`插件来实现。`InterpreterLauncher`是`Zeppelin`的一种插件类型。

### 组件之间的通信机制

`Zeppelin`前端和`Zeppelin Server`之间的通信机制主要有`Rest api`和`WebSocket`两种。`Zeppelin Server`和`Zeppelin Interpreter`是通过`Thrift RPC`来通信，而且他们彼此之间是双向通信，`Zeppelin Server`可以向`Zeppelin Interpreter`发送请求，`Zeppelin Interpreter`也可以向`Zeppelin Server`发送请求。



## 源码编译安装Apache Zeppelin

`Apache Zeppelin`官方提供了源码包和二进制包，可以根据自己的需要下载相关的包，二进制包是包含所有的解释器，如果你只需要用到其中的几种可以通过源码编译来定制化安装。

```bash
mvn clean package -DskipTests -Phadoop-2.6 -Dhadoop.version=2.6.0 -P build-distr -Dhbase.hbase.version=1.2.0 -Dhbase.hadoop.version=2.6.0
```

## 二进制安装Apache Zeppelin

~~~shell
[hadoop@hadoop001 sourcecode]$ tar -zxvf zeppelin-0.8.2-bin-all.tgz -C ~/app/
#修改配置文件
[hadoop@hadoop001 conf]$ pwd
/home/hadoop/app/zeppelin-0.8.2-bin-all/conf
[hadoop@hadoop001 conf]$ cp zeppelin-env.sh.template zeppelin-env.sh
[hadoop@hadoop001 conf]$ vim zeppelin-env.sh
export JAVA_HOME=/usr/java/jdk1.8.0_144
export HADOOP_CONF_DIR=/home/hadoop/app/hadoop-2.6.0-cdh5.7.0/etc/hadoop
export MASTER=yarn
export SPARK_HOME=/home/hadoop/app/spark-2.4.2-bin-2.6.0-cdh5.7.0
export SPARK_SUBMIT_OPTIONS="--jars /home/hadoop/lib/mysql-connector-java-5.1.46-bin.jar"
export FLINK_HOME=/home/hadoop/app/flink-1.7.2
export HBASE_HOME=/home/hadoop/app/hbase-1.2.0-cdh5.7.0

# 修改端口，主机名
[hadoop@hadoop001 conf]$ cp zeppelin-site.xml.template zeppelin-site.xml
[hadoop@hadoop001 conf]$vim zeppelin-site.xml
<property>
  <name>zeppelin.server.addr</name>
  <value>192.168.100.111</value>
  <description>Server binding address</description>
</property>

<property>
  <name>zeppelin.server.port</name>
  <value>9090</value>
  <description>Server port.</description>
</property>

~~~



### 启动Zeppelin的用户认证

`Apache Zeppelin`默认是以匿名用户访问的，即没有用户权限要求，如要实现用户权限限制，则需要修改`zeppelin-site.xml`和`shiro`配置文件。

~~~xml
# 修改zeppelin-site.xml配置文件，将以下配置项中的true改成false
<property>
  <name>zeppelin.anonymous.allowed</name>
  <value>false</value>
  <description>Anonymous user allowed by default</description>
</property>

# 复制conf目录下的shiro.ini.template 为 shiro.ini , 将shiro.ini 的[user]块中的内容修改为以下内容：
[hadoop@hadoop001 conf]$ cp shiro.ini.template shiro.ini
[hadoop@hadoop001 conf]$ vim shiro.ini
[users]
admin = admin
user1 = password2, role1, role2
user2 = password3, role3
user3 = password4, role2
.
.
.
[urls]
/api/version = anon
#/api/interpreter/setting/restart/** = authc, roles[admin,hive]
#/api/interpreter/** = authc, roles[admin,hive]
#/api/configurations/** = authc, roles[admin,hive]
#/api/credential/** = authc, roles[admin,hive]
#/** = anon
/** = authc
~~~





### 启动Apache Zeppelin

~~~shell
[hadoop@hadoop001 ~]$ cd app/zeppelin-0.8.2-bin-all/
[hadoop@hadoop001 zeppelin-0.8.2-bin-all]$ bin/zeppelin-daemon.sh start
Log dir doesn't exist, create /home/hadoop/app/zeppelin-0.8.2-bin-all/logs
Pid dir doesn't exist, create /home/hadoop/app/zeppelin-0.8.2-bin-all/run
Zeppelin start                                             [  OK  ]
[hadoop@hadoop001 zeppelin-0.8.2-bin-all]$ 
~~~



![1579366748622](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1579366748622.png)



## 创建 mysql notebook

![1579366560697](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1579366560697.png)



![1580801257905](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580801257905.png)



- 往下拉，选择` jdbc`

![1580812726923](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580812726923.png)



![1580812744945](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580812744945.png)

需要填写的有`5`个地方

~~~mysql
default.driver
default.password
default.url
default.user
artifact
~~~



### 查询mysql里面的数据



![1580812664428](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580812664428.png)



![1580819103428](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580819103428.png)



![1580819119043](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580819119043.png)



![1580819147583](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580819147583.png)



![1580819179539](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580819179539.png)



![1580819198211](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580819198211.png)



### Apache Zepplin 对接Apache Spark



![1580822737177](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580822737177.png)



~~~
当出现 java.lang.NoSuchMethodError: io.netty.buffer.PooledByteBufAllocator.defaultNumHeapArena()I
~~~

 **解决办法：**

~~~shell
# cd $ZEPPELIN_HOME/lib  
# mv netty-all-4.0.23.Final.jar netty-all-4.0.23.Final.jar_bak  
# cp $SPARK_HOME/jars/netty-all-4.1.17.Final.jar  $ZEPPELIN_HOME/lib 
~~~



![1580823253580](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580823253580.png)

>当出现 `com.fasterxml.jackson.databind.JsonMappingException: Incompatible Jackson version: 2.8.11-1`

**解决办法：**

~~~shell
# cd $ZEPPELIN_HOME/lib  
# mv jackson-databind-2.8.11.1.jar jackson-databind-2.8.11.1.jar_bak  
# cp $SPARK_HOME/jars/jackson-databind-2.6.7.1.jar $ZEPPELIN_HOME/lib  
~~~

### Spark 样例代码

![1580831909426](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580831909426.png)

### Spark SQL

![1580832112436](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580832112436.png)

![1580832172246](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580832172246.png)

### SparkSQL处理HDFS上的数据

![1580833605183](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580833605183.png)

`Apark Zeppelin`里面自带`Apache  Spark`，并且自带了`sc`，用 `Apache Zeppelin`还是比直接用`Spark Shell`或者`Spark SQL `来调试方便不少。

### Apache Zeppelin 操作Shell

![1580835973951](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580835973951.png)



### Apache Zeppelin 操作MarkDown

`Apache Zeppelin` 里面自带了 `MarkDown`

![1580840136006](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580840136006.png)

### Apache Zeppelin 集成 Apache Flink

![1580910582254](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580910582254.png)



`Zeppelin`将创建`6`个变量来表示`flink`的入口点

>- `senv` (StreamExecutionEnvironment),
>- `benv` (ExecutionEnvironment)
>- `stenv` (StreamTableEnvironment for blink planner)
>- `btenv` (BatchTableEnvironment for blink planner)
>- `stenv_2` (StreamTableEnvironment for flink planner)
>- `btenv_2` (BatchTableEnvironment for flink planner)





### Apache Zeppelin 集成 Apache HBase

~~~shell
[hadoop@hadoop001 hbase]$ pwd
/home/hadoop/app/zeppelin-0.8.2-bin-all/interpreter/hbase
[hadoop@hadoop001 hbase]$ mkdir hbase_jar
[hadoop@hadoop001 hbase]$ mv hbase*.jar hbase_jar/
[hadoop@hadoop001 hbase]$ mv hadoop*.jar hbase_jar/
[hadoop@hadoop001 hbase]$ mv zookeeper-3.4.6.jar hbase_jar/
[hadoop@hadoop001 hbase]$ 
[hadoop@hadoop001 hbase]$ cp -f /home/hadoop/app/hbase-1.2.0-cdh5.7.0/lib/*.jar /home/hadoop/app/zeppelin-0.8.2-bin-all/interpreter/hbase/
~~~

- 编辑`interpreter.json`，位置`/home/hadoop/app/zeppelin-0.8.2-bin-all/interpreter/hbase/interpreter.json`,修改`hbase.home`。

```xml
  "hbase.home": {
        "name": "hbase.home",
        "value": "/home/hadoop/app/hbase-1.2.0-cdh5.7.0/",
        "type": "string"
      },
```

- 重启 `Apache Zeppelin`

  ```shell
  cd /home/hadoop/app/zeppelin-0.8.2-bin-all/bin
  zeppelin-daemon.sh restart
  ```

- 测试

![1581046124420](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581046124420.png)



### Apache Zeppelin 集成 HDFS

![1580880512706](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1580880512706.png)



![1581142915393](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581142915393.png)



### Apache Zeppelin 集成 Apache Hive

- 添加hive的Interpreters

![1581053916323](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581053916323.png)

![1581053978266](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581053978266.png)

![1581054050294](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581054050294.png)

- 启动hive2 

> [hadoop@hadoop001 lib]$ hiveserver2 &

- 测试

![1581053794558](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581053794558.png)



### Apache Zeppelin 集成 Elasticsearch

![1581143970232](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581143970232.png)

- 可以直接用 Elasticsearch 的 Interpreters

![1581144076840](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581144076840.png)



- 也可以直接用 shell 的 Interpreters

![1581144117200](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1581144117200.png)

## Apache Zeppelin  VS  Jupyter Notebook

Jupyter Notebook是IPython Notebook的演变版，更出名。

| 对比       | Zeppelin | Jupyter  |
| ---------- | -------- | -------- |
| 开发语言   | python   | java     |
| GithubStar | 4.6k     | 6.8k     |
| 安装       | 简单     | 简单     |
| 诞生       | 2012年   | 2013年   |
| 支持Spark  | 支持     | 支持     |
| 支持Flink  | 支持     | 暂不支持 |
| 多用户     | 支持     | 不支持   |
| 权限       | 支持     | 不支持   |



Apache Zeppelin 功能强大，适合于企业级应用，而Jupyter 更适用于个人测试学习，Apache Zeppelin集成了Flink/Spark等大数据知名产品，而Jupyter未集成Flink，功能远不如Apache Zeppelin强大。

### 参考文档

>- [http://zeppelin.apache.org/](<http://zeppelin.apache.org/>)
>- [https://www.jianshu.com/p/44b43e046b44](<https://www.jianshu.com/p/44b43e046b44>)































