---
layout:     post
title:      "hadoop lzo 压缩及测试"
date:       2019-03-30 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - hadoop,mapreduce
---






### lzo配置

 因为lzo 压缩不是hadoop自带的，需要编译安装。

    [root@hadoop001 app]# wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz
    
    [root@hadoop001 tar]# tar -zxvf lzo-2.10.tar.gz -C /home/hadoop/app/
    
    [root@hadoop001 lzo-2.10]# pwd
    /home/hadoop/app/lzo-2.10
    
    [root@hadoop001 lzo-2.10]# ./configure
    [root@hadoop001 lzo-2.10]# make install
    [root@hadoop001 app]# chown hadoop:hadoop -R lzo-2.10/
    
    



### 配置lzop

    [root@hadoop001 app]# wget http://www.lzop.org/download/lzop-1.04.tar.gz
    [root@hadoop001 app]# tar -zxvf lzop-1.04.tar.gz -C /home/hadoop/app/
    
    [root@hadoop001 lzop-1.04]# pwd
    /home/hadoop/app/lzop-1.04
    [root@hadoop001 lzop-1.04]# ./configure 
    [root@hadoop001 lzop-1.04]# make  && make install
    [root@hadoop001 app]# chown hadoop:hadoop -R lzop-1.04/
    



### 编译hadoop-lzo

    [root@hadoop001 app]# wget https://github.com/twitter/hadoop-lzo/archive/master.zip
    [root@hadoop001 app]# unzip master -d /home/hadoop/app/
    
    [root@hadoop001 hadoop-lzo-master]# pwd
    /home/hadoop/app/hadoop-lzo-master
    
    [root@hadoop001 app]# chown hadoop:hadoop -R hadoop-lzo-master/
    
    # 修改hadoop-lzo-master 中的pom.xml文件
    #在89行，修改hadoop的版本
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <hadoop.current.version>2.6.0-cdh5.7.0</hadoop.current.version>    
        <hadoop.old.version>1.0.4</hadoop.old.version>
    </properties>
    
    # 在pom文件中加上仓库，在85行左右
    <repository>
       <id>cloudera</id>
       <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
    </repository>
    
    
    # 编译
    [hadoop@hadoop001 hadoop-lzo-master]$C_INCLUDE_PATH=/usr/local/include 
    [hadoop@hadoop001 hadoop-lzo-master]$LIBRARY_PATH=/usr/local/lib
    [hadoop@hadoop001 hadoop-lzo-master]$  mvn clean package -Dmaven.test.skip=true
    
    # 拷贝相关文件到本地库
    [hadoop@hadoop001 hadoop-lzo-master]$ cd target/native/Linux-amd64-64/
    [hadoop@hadoop001 Linux-amd64-64]$ tar -cBf - -C lib . | tar -xBvf - -C ~
    [hadoop@hadoop001 ~]$ cp ~/libgplcompression* /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/lib/native/
    [hadoop@hadoop001 target]$ cp hadoop-lzo-0.4.21-SNAPSHOT.jar /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/share/hadoop/common/



### 配置core-site.xml

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
        <!--支持LZO使用类-->
     <property>
         <name>io.compression.codec.lzo.class</name>
         <value>com.hadoop.compression.lzo.LzopCodec</value>
     </property>
    



### 配置mapred-site.xml

    <!--启用map中间压缩类-->
    <property>
       <name>mapred.map.output.compression.codec</name>
       <value>com.hadoop.compression.lzo.LzopCodec</value>
    </property>
    <!--启用mapreduce文件压缩-->
    <property>
        <name>mapreduce.output.fileoutputformat.compress</name>
        <value>true</value>
    </property> 
    <!--启用mapreduce压缩类-->
    <property>
       <name>mapreduce.output.fileoutputformat.compress.codec</name>
       <value>com.hadoop.compression.lzo.LzopCodec</value>
    </property>
    <!--配置Jar包-->
    <property>
        <name>mapred.child.env</name>
        <value>LD_LIBRARY_PATH=/usr/local/lib</value>
    </property>
    



### lzo测试

    # 压缩前1.7GB
    [hadoop@hadoop001 data]$ du -sh data.log
    1.7G	data.log
    [hadoop@hadoop001 data]$ lzop data.log
    [hadoop@hadoop001 data]$ du -sh data.log.lzo 
    811M	data.log.lzo
    
    [hadoop@hadoop001 data]$ hdfs dfs -mkdir -p /hadoop/compress
    [hadoop@hadoop001 data]$ hdfs dfs -put data.log.lzo /hadoop/compress
    
    hadoop@hadoop001 lib]$ hadoop jar g6-hadoop-1.0.jar com.ruozedata.hadoop.mapreduce.driver.LogETLDriver /hadoop/compress /hadoop/compress/output
    19/03/30 03:35:36 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
    19/03/30 03:35:37 WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
    19/03/30 03:35:37 INFO input.FileInputFormat: Total input paths to process : 1
    19/03/30 03:35:37 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
    19/03/30 03:35:37 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev f1deea9a313f4017dd5323cb8bbb3732c1aaccc5]
    19/03/30 03:35:37 INFO mapreduce.JobSubmitter: number of splits:1
    
    # 注意：没有指定索引的时候，lzo 只会出现一个分片，number of splits:1
    
    # 不分片的时候
    [hadoop@hadoop001 lib]$ hadoop fs -du -s -h  /hadoop/compress/output/part-r-00000.lzo
    769.5 M  769.5 M  /hadoop/compress/output/part-r-00000.lzo
    [hadoop@hadoop001 lib]$ 
    





### 指定分片

    [hadoop@hadoop001 lib]$ hdfs dfs -mkdir -p /hadoop/compress_index
    [hadoop@hadoop001 lib]$ hdfs dfs -put /home/hadoop/data/data.log.lzo /hadoop/compress_index
    
    
### 创建索引

    # 查找创建索引的jar包
    [root@hadoop001 hadoop]# find / -name hadoop-lzo*
    /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/share/hadoop/common/hadoop-lzo-0.4.21-SNAPSHOT.jar
    /home/hadoop/app/hadoop-lzo-master
    /home/hadoop/app/hadoop-lzo-master/target/classes/hadoop-lzo-build.properties
    /home/hadoop/app/hadoop-lzo-master/target/hadoop-lzo-0.4.21-SNAPSHOT-javadoc.jar
    /home/hadoop/app/hadoop-lzo-master/target/hadoop-lzo-0.4.21-SNAPSHOT-sources.jar
    /home/hadoop/app/hadoop-lzo-master/target/hadoop-lzo-0.4.21-SNAPSHOT.jar
    
    
    
    [hadoop@hadoop001 ~]$ hadoop jar /home/hadoop/app/hadoop-lzo-master/target/hadoop-lzo-0.4.21-SNAPSHOT.jar com.hadoop.compression.lzo.DistributedLzoIndexer /hadoop/compress_index
    
    [hadoop@hadoop001 ~]$ hdfs dfs -lsr /hadoop/compress_index
    lsr: DEPRECATED: Please use 'ls -R' instead.
    -rw-r--r--   1 hadoop supergroup  849764876 2019-03-30 03:51 /hadoop/compress_index/data.log.lzo
    -rw-r--r--   1 hadoop supergroup      53560 2019-03-30 11:37 /hadoop/compress_index/data.log.lzo.index
    [hadoop@hadoop001 ~]$ 
    
    


### 执行MapReduce

    # 需要修改MR的代码，在diver中添加如下代码
    LzoTextInputFormat.setInputPaths(job,new Path(input));
    
    [hadoop@hadoop001 lib]$ hadoop jar g6-hadoop-1.0.jar com.ruozedata.hadoop.mapreduce.driver.LogETLDirverLzo /hadoop/compress_index/ /hadoop/compress_index/output
    19/03/30 13:15:41 INFO driver.LogETLDirverLzo: Processing trade with value: /hadoop/compress_index/output  
    19/03/30 13:15:41 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
    19/03/30 13:15:41 WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
    19/03/30 13:15:42 INFO input.FileInputFormat: Total input paths to process : 2
    19/03/30 13:15:42 INFO mapreduce.JobSubmitter: number of splits:7
    
    # 指定了lzo的索引，811M / 128 = 6.33 > 6.1 会出现7个分片


### 总结
 
 没有指定索引的时候，lzo 只会出现一个分片，number of splits:1 ; 指定了lzo的索引，number of splits:7 ; 811M / 128 = 6.33 > 6.1 会出现7个分片。
 
 [MapReduce分片划分的策略](https://justdodt.github.io/2018/04/09/MapReduce%E5%88%86%E7%89%87/)






