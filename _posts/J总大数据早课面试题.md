# 早课
#### 2019/03/21
> yarn资源调优怎么调，依据是什么,比如服务器256G物理内存,有DN NM RS三个进程,yarn资源调优怎么做  
答:RS(RegionServer)是HBase的进程,一般经验值放30G  
从两个角度回答:  
1.机器总内存  ，预留内存，各个进程内存的经验值  
2.余下就是yarn资源的总内存  

```
<property>
<name>yarn.nodemanager.resource.cpu-vcores</name>
<value>8</value>
</property>
 
<property>
<name>yarn.nodemanager.resource.memory-mb</name>
<value>8192</value>
</property>
 
<property>
<name>yarn.scheduler.minimum-allocation-mb</name>
<value>1024</value>
</property>
 
<property>
<name>yarn.scheduler.maximum-allocation-mb</name>
<value>8192</value>
</property>
```
#### 2019/03/27
> 谈谈HBase的分割和合并（小合并，大合并）?


#### 2019/03/28
> 1.谈谈HBase在生产如何优雅下线或重启RS节点?


> 2.有一个cluster,2个nn,8个dn,想缩减为1个dn,如何操作(分CDH跟原生Hadoop)


#### 2019/03/29
> 1.HBase的hbck命令使用过吗？具体说说，带场景维护

> 2.实时写 去重启hbase，如何保证数据不丢呢


#### 2019/04/02
> HBase生产上，你认为要监控哪些指标?怎样做监控?怎样做预警?

#### 2019/04/04
> 1.小文件有什么危害?  
> 2.如何规避小文件产生?  
> 3.真产生小文件，怎样处理?  
> 从hive和spark到hdfs两个方向

某群友答案：

危害：
1、增加namenode压力、Namenode相关性能故障频发
2、浪费yarn计算资源、以至于降低计算效率

规避：
1、合理根据业务优化入库数据，合理设置分区，尤其是流式数据，避免大量小文件分片入库
2、合理设置map、reduce splitsize，考虑输入合并、输出合并、中间结果压缩 

处理：
1、业务优化很关键。
2、历史不访问的数据进行har压缩
3、离线分析fsimage，找出小文件重灾区，写出合并小文件工具（Map读入，较少reduce输出）



HBase有小文件吗？



#### 2019/04/10

Spark 2.x 内存管理哪几个类文件，说说你的理解。



#### 2019/04/12

![1555058346212](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1555058346212.png)



load 一般不超过10；1min  5min  15min代表的值

值越大 代表系统越繁忙，有可能夯住，spark作业一般不会超过十，长服务可能时间长了会破百。

系统负载三个数字的含义
一般来说，系统每隔5秒钟，会检查一下当前系统活跃的进程数。这个活跃进程要满足3个前提 
* 它没有在等待I/O操作的结果 
* 它没有主动进入等待状态(也就是没有调用’wait’) 
* 没有被停止(例如：等待终止) 
  而系统负载的三个数值分别表示的是1分钟，5分钟和15分钟系统负载的平均值。 
  对于一个具有n核处理器的系统来说，当系统负载的load average为n的时候，表示系统差不多刚刚好满负荷，但是已经没有额外的经历去处理其它任务了。当load average 大于n的时候，表示系统超负荷运转。一般来说为了使系统能正常运转，我们经验上，任务load average / n < 0.7 是一般能接受的情况。

首先在命令行中输入top 命令，然后按数字1，既可以看见逻辑CPU的颗数。

~~~shell
[root@hadoop001 ~]# top
top - 22:48:19 up 5 days, 12 min,  6 users,  load average: 0.00, 0.00, 0.00
Tasks: 164 total,   1 running, 163 sleeping,   0 stopped,   0 zombie
Cpu0  :  0.3%us,  0.3%sy,  0.0%ni, 99.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu1  :  0.0%us,  0.3%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu2  :  0.0%us,  0.3%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu3  :  0.3%us,  0.3%sy,  0.0%ni, 99.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:   4040896k total,  3542400k used,   498496k free,   127032k buffers
Swap:  2097144k total,   147992k used,  1949152k free,  1310448k cached

   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                            
131062 hadoop    20   0 2918m 455m  18m S  2.3 11.5  12:03.29 java                                                                               
  7811 root      20   0 15032 1276  948 R  0.7  0.0   0:02.45 top  
~~~



mapreduce 和 spark跑集群的时候，一定要分队列跑







