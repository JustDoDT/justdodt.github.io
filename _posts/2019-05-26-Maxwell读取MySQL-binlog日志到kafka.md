---
layout:     post
title:      "Maxwell读取MySQL binlog日志到Kafka"
date:       2019-05-26 02:41:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - Maxwell

---



### 启动MySQL

###创建maxwell的数据库和用户

### 在MySQL中创建一个测试数据库和表

前面三个步骤详见 [Maxwell读取MySQL binlog日志通过stdout展示](<https://justdodt.github.io/2019/05/25/Maxwell%E8%AF%BB%E5%8F%96MySQL-binlog%E6%97%A5%E5%BF%97%E9%80%9A%E8%BF%87stdout%E5%B1%95%E7%A4%BA/>)

### 启动Zookeeper

~~~
[hadoop@hadoop001 ~]$ cd $ZK_HOME/bin
[hadoop@hadoop001 bin]$ ./zkServer.sh start
~~~

### 启动kafka，并创建主题为maxwell的topic

~~~
[hadoop@hadoop001 bin]$ cd $KAFKA_HOME
//查看kafka版本，防止maxwell不支持
[hadoop@hadoop001 kafka]$ find ./libs/ -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*'
kafka_2.11-0.10.0.1-sources.jar
//启动kafka-server服务
[hadoop@hadoop001 kafka]$ nohup bin/kafka-server-start.sh config/server.properties &
[hadoop@hadoop001 kafka]$ jps
13460 QuorumPeerMain
14952 Jps
13518 Kafka
[hadoop@hadoop001 kafka]$ bin/kafka-topics.sh --create --zookeeper 192.168.137.2:2181/kafka --replication-factor 1 --partitions 3 --topic maxwell
Created topic "maxwell".
[hadoop@hadoop001 kafka]$ bin/kafka-topics.sh --list --zookeeper 192.168.137.2:2181/kafka
__consumer_offsets
maxwell
~~~



### 启动kafaka的消费者，检查数据是否到位

~~~
[hadoop@hadoop001 kafka]$ bin/kafka-console-consumer.sh --zookeeper 192.168.137.2:2181/kafka --topic maxwell --from-beginning
~~~



### 启动maxwell进程

~~~
//先检查maxwell是否支持kafka-0.10.0.1
[root@hadoop000 ~]# cd /root/app/maxwell-1.17.1/lib/kafka-clients
[root@hadoop001 kafka-clients]# ll
total 5556
-rw-r--r-- 1 yarn games  746207 Jul  3  2018 kafka-clients-0.10.0.1.jar
-rw-r--r-- 1 yarn games  951041 Jul  3  2018 kafka-clients-0.10.2.1.jar
-rw-r--r-- 1 yarn games 1419544 Jul  3  2018 kafka-clients-0.11.0.1.jar
-rw-r--r-- 1 yarn games  324016 Jul  3  2018 kafka-clients-0.8.2.2.jar
-rw-r--r-- 1 yarn games  641408 Jul  3  2018 kafka-clients-0.9.0.1.jar
-rw-r--r-- 1 yarn games 1591338 Jul  3  2018 kafka-clients-1.0.0.jar
//发现支持kafka-0.10.0.1版本，假如没有生产上正在用的kafka版本的jar包，可以直接把这个版本的client jar包copy进来
//启动maxwell
[root@hadoop001 maxwell-1.17.1]# bin/maxwell --user='maxwell' --password='maxwell' --host='127.0.0.1' --producer=kafka --kafka_version=0.10.0.1 --kafka.bootstrap.servers=192.168.137.2:9092 --kafka_topic=maxwell
Using kafka version: 0.10.0.1
10:16:52,979 WARN  MaxwellMetrics - Metrics will not be exposed: metricsReportingType not configured.
10:16:53,451 INFO  ProducerConfig - ProducerConfig values: 
        metric.reporters = []
        metadata.max.age.ms = 300000
        reconnect.backoff.ms = 50
        sasl.kerberos.ticket.renew.window.factor = 0.8
        bootstrap.servers = [192.168.137.2:9092]
        ssl.keystore.type = JKS
        sasl.mechanism = GSSAPI
        max.block.ms = 60000
        interceptor.classes = null
        ssl.truststore.password = null
        client.id = 
        ssl.endpoint.identification.algorithm = null
        request.timeout.ms = 30000
        acks = 1
        receive.buffer.bytes = 32768
        ssl.truststore.type = JKS
        retries = 0
        ssl.truststore.location = null
        ssl.keystore.password = null
        send.buffer.bytes = 131072
        compression.type = none
        metadata.fetch.timeout.ms = 60000
        retry.backoff.ms = 100
        sasl.kerberos.kinit.cmd = /usr/bin/kinit
        buffer.memory = 33554432
        timeout.ms = 30000
        key.serializer = class org.apache.kafka.common.serialization.StringSerializer
        sasl.kerberos.service.name = null
        sasl.kerberos.ticket.renew.jitter = 0.05
        ssl.trustmanager.algorithm = PKIX
        block.on.buffer.full = false
        ssl.key.password = null
        sasl.kerberos.min.time.before.relogin = 60000
        connections.max.idle.ms = 540000
        max.in.flight.requests.per.connection = 5
        metrics.num.samples = 2
        ssl.protocol = TLS
        ssl.provider = null
        ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
        batch.size = 16384
        ssl.keystore.location = null
        ssl.cipher.suites = null
        security.protocol = PLAINTEXT
        max.request.size = 1048576
        value.serializer = class org.apache.kafka.common.serialization.StringSerializer
        ssl.keymanager.algorithm = SunX509
        metrics.sample.window.ms = 30000
        partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
        linger.ms = 0

10:16:53,512 INFO  ProducerConfig - ProducerConfig values: 
        metric.reporters = []
        metadata.max.age.ms = 300000
        reconnect.backoff.ms = 50
        sasl.kerberos.ticket.renew.window.factor = 0.8
        bootstrap.servers = [192.168.137.2:9092]
        ssl.keystore.type = JKS
        sasl.mechanism = GSSAPI
        max.block.ms = 60000
        interceptor.classes = null
        ssl.truststore.password = null
        client.id = producer-1
        ssl.endpoint.identification.algorithm = null
        request.timeout.ms = 30000
        acks = 1
        receive.buffer.bytes = 32768
        ssl.truststore.type = JKS
        retries = 0
        ssl.truststore.location = null
        ssl.keystore.password = null
        send.buffer.bytes = 131072
        compression.type = none
        metadata.fetch.timeout.ms = 60000
        retry.backoff.ms = 100
        sasl.kerberos.kinit.cmd = /usr/bin/kinit
        buffer.memory = 33554432
        timeout.ms = 30000
        key.serializer = class org.apache.kafka.common.serialization.StringSerializer
        sasl.kerberos.service.name = null
        sasl.kerberos.ticket.renew.jitter = 0.05
        ssl.trustmanager.algorithm = PKIX
        block.on.buffer.full = false
        ssl.key.password = null
        sasl.kerberos.min.time.before.relogin = 60000
        connections.max.idle.ms = 540000
        max.in.flight.requests.per.connection = 5
        metrics.num.samples = 2
        ssl.protocol = TLS
        ssl.provider = null
        ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
        batch.size = 16384
        ssl.keystore.location = null
        ssl.cipher.suites = null
        security.protocol = PLAINTEXT
        max.request.size = 1048576
        value.serializer = class org.apache.kafka.common.serialization.StringSerializer
        ssl.keymanager.algorithm = SunX509
        metrics.sample.window.ms = 30000
        partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
        linger.ms = 0

10:16:53,516 INFO  AppInfoParser - Kafka version : 0.10.0.1
10:16:53,516 INFO  AppInfoParser - Kafka commitId : a7a17cdec9eaa6c5
10:16:53,550 INFO  Maxwell - Maxwell v1.17.1 is booting (MaxwellKafkaProducer), starting at Position[BinlogPosition[mysql-bin.000016:116360], lastHeartbeat=1552092988288]
10:16:53,730 INFO  MysqlSavedSchema - Restoring schema id 1 (last modified at Position[BinlogPosition[mysql-bin.000014:5999], lastHeartbeat=0])
10:16:53,846 INFO  BinlogConnectorReplicator - Setting initial binlog pos to: mysql-bin.000016:116360
10:16:53,951 INFO  BinaryLogClient - Connected to 127.0.0.1:3306 at mysql-bin.000016/116360 (sid:6379, cid:4)
10:16:53,951 INFO  BinlogConnectorLifecycleListener - Binlog connected.
~~~



### 在MySQL中更新一条数据

~~~
mysql> update emp set sal=502 where empno=6001;
mysql> update emp set sal=603 where empno=6001;
mysql> create table emp1 select * from emp;
~~~



### 查看kafka的消费者

~~~
//对应第一条insert语句
{"database":"hlwtest","table":"emp","type":"update","ts":1552097863,"xid":89,"commit":true,"data":{"empno":6001,"ename":"SIWA","job":"DESIGNER","mgr":7001,"hiredate":"2019-03-08 00:00:00","sal":502.00,"comm":6000.00,"deptno":40},"old":{"sal":501.00}}
//对应第二条insert语句
{"database":"hlwtest","table":"emp","type":"update","ts":1552097951,"xid":123,"commit":true,"data":{"empno":6001,"ename":"SIWA","job":"DESIGNER","mgr":7001,"hiredate":"2019-03-08 00:00:00","sal":603.00,"comm":6000.00,"deptno":40},"old":{"sal":502.00}}
//对应建表，相当于在新表emp1里插入了emp表里的所有数据
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":0,"data":{"empno":6001,"ename":"SIWA","job":"DESIGNER","mgr":7001,"hiredate":"2019-03-08 00:00:00","sal":603.00,"comm":6000.00,"deptno":40}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":1,"data":{"empno":7369,"ename":"SMITH","job":"CLERK","mgr":7902,"hiredate":"1980-12-17 00:00:00","sal":800.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":2,"data":{"empno":7499,"ename":"ALLEN","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-20 00:00:00","sal":1600.00,"comm":300.00,"deptno":30}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":3,"data":{"empno":7521,"ename":"WARD","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-22 00:00:00","sal":1250.00,"comm":500.00,"deptno":30}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":4,"data":{"empno":7566,"ename":"JONES","job":"MANAGER","mgr":7839,"hiredate":"1981-04-02 00:00:00","sal":2975.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":5,"data":{"empno":7654,"ename":"MARTIN","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-28 00:00:00","sal":1250.00,"comm":1400.00,"deptno":30}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":6,"data":{"empno":7698,"ename":"BLAKE","job":"MANAGER","mgr":7839,"hiredate":"1981-05-01 00:00:00","sal":2850.00,"comm":0.00,"deptno":30}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":7,"data":{"empno":7782,"ename":"CLARK","job":"MANAGER","mgr":7839,"hiredate":"1981-06-09 00:00:00","sal":2450.00,"comm":0.00,"deptno":10}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":8,"data":{"empno":7788,"ename":"SCOTT","job":"ANALYST","mgr":7566,"hiredate":"1987-04-19 00:00:00","sal":3000.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":9,"data":{"empno":7839,"ename":"KING","job":"PRESIDENT","mgr":0,"hiredate":"1981-11-17 00:00:00","sal":5000.00,"comm":0.00,"deptno":10}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":10,"data":{"empno":7844,"ename":"TURNER","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-08 00:00:00","sal":1500.00,"comm":0.00,"deptno":30}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":11,"data":{"empno":7876,"ename":"ADAMS","job":"CLERK","mgr":7788,"hiredate":"1987-05-23 00:00:00","sal":1100.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":12,"data":{"empno":7900,"ename":"JAMES","job":"CLERK","mgr":7698,"hiredate":"1981-12-03 00:00:00","sal":950.00,"comm":0.00,"deptno":30}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":13,"data":{"empno":7902,"ename":"FORD","job":"ANALYST","mgr":7566,"hiredate":"1981-12-03 00:00:00","sal":3000.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"xoffset":14,"data":{"empno":7934,"ename":"MILLER","job":"CLERK","mgr":7782,"hiredate":"1982-01-23 00:00:00","sal":1300.00,"comm":0.00,"deptno":10}}
{"database":"hlwtest","table":"emp1","type":"insert","ts":1552098958,"xid":435,"commit":true,"data":{"empno":8888,"ename":"HIVE","job":"PROGRAM","mgr":7839,"hiredate":"1988-01-23 00:00:00","sal":10300.00,"comm":0.00,"deptno":null}}
~~~



`数据已经正常同步到kafka中`

### Maxwell的过滤功能

参考过滤配置： [<http://maxwells-daemon.io/filtering/>](<http://maxwells-daemon.io/filtering/>)

~~~
[root@hadoop001 maxwell-1.17.1]# bin/maxwell --user='maxwell' --password='maxwell' \
--host='127.0.0.1' --filter 'exclude: *.*, include:hlwtest.emp1' \
--producer=kafka --kafka_version=0.10.0.1 --kafka.bootstrap.servers=192.168.137.2:9092 --kafka_topic=maxwell
~~~



`--filter 'exclude: *.*, include:hlwtest.emp1'的意思是只监控hlwtest.emp1表的变化，其他的都不监控`

~~~
//MySQL中update数据
mysql> update emp set sal=730 where empno=6001;
mysql> update emp1 set sal=330 where empno=6001;
mysql> update emp set sal=730 where empno=6001;
mysql> update emp1 set sal=331 where empno=6001;

//Kafka消费者接收到的数据
{"database":"hlwtest","table":"emp1","type":"update","ts":1552099858,"xid":916,"commit":true,"data":{"empno":6001,"ename":"SIWA","job":"DESIGNER","mgr":7001,"hiredate":"2019-03-08 00:00:00","sal":330.00,"comm":6000.00,"deptno":40},"old":{"sal":321.00}}
{"database":"hlwtest","table":"emp1","type":"update","ts":1552099858,"xid":922,"commit":true,"data":{"empno":6001,"ename":"SIWA","job":"DESIGNER","mgr":7001,"hiredate":"2019-03-08 00:00:00","sal":331.00,"comm":6000.00,"deptno":40},"old":{"sal":330.00}}
~~~



`确实只消费到了emp1表的update语句，而没有接收到emp表的更新`

### Maxwell 的bootstrap

~~~
mysql> insert into maxwell.bootstrap (database_name, table_name) values ("hlwtest", "emp");
mysql> select * from maxwell.bootstrap;
+----+---------------+------------+--------------+-------------+---------------+------------+------------+---------------------+---------------------+------------------+-----------------+-----------+
| id | database_name | table_name | where_clause | is_complete | inserted_rows | total_rows | created_at | started_at          | completed_at        | binlog_file      | binlog_position | client_id |
+----+---------------+------------+--------------+-------------+---------------+------------+------------+---------------------+---------------------+------------------+-----------------+-----------+
|  1 | hlwtest       | emp        | NULL         |           1 |            16 |          0 | NULL       | 2019-03-09 11:33:11 | 2019-03-09 11:33:11 | mysql-bin.000018 |          225498 | maxwell   |
+----+---------------+------------+--------------+-------------+---------------+------------+------------+---------------------+---------------------+------------------+-----------------+-----------+
~~~



### kafka的消费窗口

~~~
{"database":"maxwell","table":"bootstrap","type":"insert","ts":1552102248,"xid":1555,"commit":true,"data":{"id":1,"database_name":"hlwtest","table_name":"emp","where_clause":null,"is_complete":0,"inserted_rows":0,"total_rows":0,"created_at":null,"started_at":null,"completed_at":null,"binlog_file":null,"binlog_position":0,"client_id":"maxwell"}}
{"database":"hlwtest","table":"emp","type":"bootstrap-start","ts":1552102391,"data":{}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":6001,"ename":"SIWA","job":"DESIGNER","mgr":7001,"hiredate":"2019-03-08 00:00:00","sal":730.00,"comm":6000.00,"deptno":40}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7369,"ename":"SMITH","job":"CLERK","mgr":7902,"hiredate":"1980-12-17 00:00:00","sal":800.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7499,"ename":"ALLEN","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-20 00:00:00","sal":1600.00,"comm":300.00,"deptno":30}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7521,"ename":"WARD","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-22 00:00:00","sal":1250.00,"comm":500.00,"deptno":30}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7566,"ename":"JONES","job":"MANAGER","mgr":7839,"hiredate":"1981-04-02 00:00:00","sal":2975.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7654,"ename":"MARTIN","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-28 00:00:00","sal":1250.00,"comm":1400.00,"deptno":30}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7698,"ename":"BLAKE","job":"MANAGER","mgr":7839,"hiredate":"1981-05-01 00:00:00","sal":2850.00,"comm":0.00,"deptno":30}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7782,"ename":"CLARK","job":"MANAGER","mgr":7839,"hiredate":"1981-06-09 00:00:00","sal":2450.00,"comm":0.00,"deptno":10}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7788,"ename":"SCOTT","job":"ANALYST","mgr":7566,"hiredate":"1987-04-19 00:00:00","sal":3000.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7839,"ename":"KING","job":"PRESIDENT","mgr":0,"hiredate":"1981-11-17 00:00:00","sal":5000.00,"comm":0.00,"deptno":10}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7844,"ename":"TURNER","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-08 00:00:00","sal":1500.00,"comm":0.00,"deptno":30}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7876,"ename":"ADAMS","job":"CLERK","mgr":7788,"hiredate":"1987-05-23 00:00:00","sal":1100.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7900,"ename":"JAMES","job":"CLERK","mgr":7698,"hiredate":"1981-12-03 00:00:00","sal":950.00,"comm":0.00,"deptno":30}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7902,"ename":"FORD","job":"ANALYST","mgr":7566,"hiredate":"1981-12-03 00:00:00","sal":3000.00,"comm":0.00,"deptno":20}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":7934,"ename":"MILLER","job":"CLERK","mgr":7782,"hiredate":"1982-01-23 00:00:00","sal":1300.00,"comm":0.00,"deptno":10}}
{"database":"hlwtest","table":"emp","type":"bootstrap-insert","ts":1552102391,"data":{"empno":8888,"ename":"HIVE","job":"PROGRAM","mgr":7839,"hiredate":"1988-01-23 00:00:00","sal":10300.00,"comm":0.00,"deptno":null}}
{"database":"maxwell","table":"bootstrap","type":"update","ts":1552102391,"xid":1617,"commit":true,"data":{"id":1,"database_name":"hlwtest","table_name":"emp","where_clause":null,"is_complete":1,"inserted_rows":16,"total_rows":0,"created_at":null,"started_at":"2019-03-09 11:33:11","completed_at":"2019-03-09 11:33:11","binlog_file":"mysql-bin.000018","binlog_position":225498,"client_id":"maxwell"},"old":{"is_complete":0,"inserted_rows":1,"completed_at":null}}
{"database":"hlwtest","table":"emp","type":"bootstrap-complete","ts":1552102391,"data":{}}
~~~





