**jps命令的真相**

    [hadoop@hadoop001 ~]$ jps
    13397 NameNode
    13718 SecondaryNameNode
    13530 DataNode
    13887 Jps
    [hadoop@hadoop001 ~]$ 
    


**位置哪里的**

    [hadoop@hadoop001 ~]$ which jps
    /usr/java/jdk1.8.0_144/bin/jps
    [hadoop@hadoop001 ~]$ 




**对应的进程的标识文件在哪   /tmp/hsperfdata_username**

    [hadoop@hadoop001 /]$ ll /tmp/ | grep hsperfdata
    drwxr-xr-x. 2 hadoop hadoop 4096 Mar 16 06:02 hsperfdata_hadoop
    drwxr-xr-x. 2 root   root   4096 Mar 16 06:01 hsperfdata_root
    [hadoop@hadoop001 /]$ 
    
    [hadoop@hadoop001 hsperfdata_hadoop]$ pwd
    /tmp/hsperfdata_hadoop
    [hadoop@hadoop001 hsperfdata_hadoop]$ ll
    total 96
    -rw-------. 1 hadoop hadoop 32768 Mar 16 06:09 13397
    -rw-------. 1 hadoop hadoop 32768 Mar 16 06:09 13530
    -rw-------. 1 hadoop hadoop 32768 Mar 16 06:09 13718
    [hadoop@hadoop001 hsperfdata_hadoop]$ 





**root用户看所有用户的jps结果，普通用户只能看自己的**

    [root@hadoop001 ~]# jps
    13397 NameNode
    13718 SecondaryNameNode
    13530 DataNode
    14028 Jps
    
    [root@hadoop001 home]# su - haha
    [haha@hadoop001 ~]$ jps
    14005 Jps
    [haha@hadoop001 ~]$ 
    



**process information unavailable**

`真假判断: ps -ef|grep namenode 真正判断进程是否可用`

**当出现process information unavailable ,不一定表示不可以用**

    [root@hadoop001 ~]# jps
    1520 Jps
    1378 -- process information unavailable
    1210 -- process information unavailable
    1086 -- process information unavailable





**kill: 可能是人为的，也可能是某个进程在Linux系统中出现(OOM)自动给你kill**

    [root@hadoop001 tmp]# rm -rf hsperfdata_hadoop
    [root@hadoop001 tmp]# 
    [root@hadoop001 tmp]# jps
    1906 Jps
    [root@hadoop001 tmp]# 
    
    
    
    **pid文件 集群进程启动和停止要的文件**
    
    -rw-rw-r-- 1 hadoop hadoop    5 Feb 16 20:56 hadoop-hadoop-datanode.pid
    -rw-rw-r-- 1 hadoop hadoop    5 Feb 16 20:56 hadoop-hadoop-namenode.pid
    -rw-rw-r-- 1 hadoop hadoop    5 Feb 16 20:57 hadoop-hadoop-secondarynamenode.pid
    
    
    
    

**Linux在tmp目录中会定期删除一些文件和文件夹，此周期为30天**

    [hadoop@hadoop001 hadoop]$ tail -10  hadoop-env.sh  
    
    # The directory where pid files are stored. /tmp by default.
    # NOTE: this should be set to a directory that can only be written to by 
    #       the user that will run the hadoop daemons.  Otherwise there is the
    #       potential for a symlink attack.
    export HADOOP_PID_DIR=${HADOOP_PID_DIR}
    export HADOOP_SECURE_DN_PID_DIR=${HADOOP_PID_DIR}
    
    # A string representing this instance of hadoop. $USER by default.
    export HADOOP_IDENT_STRING=$USER
    
    #比如自己修改PID文件路径为如下的方式
    [hadoop@hadoop001 hadoop]$ 
    mkdir /data/tmp
    chmod -R 777 /data/tmp
    export HADOOP_PID_DIR=/data/tmp



**在生产中，我们需要修改PID的存放路径，因为默认放在/tmp下，如果删除了PID文件，会造成进程无法启动关闭。**

    [hadoop@hadoop001 tmp]$ rm -rf hadoop-hadoop-datanode.pid
    
    [hadoop@hadoop001 tmp]$ stop-dfs.sh 
    19/03/16 06:43:23 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    Stopping namenodes on [localhost]
    localhost: stopping namenode
    hadoop001: no datanode to stop
    Stopping secondary namenodes [hadoop001]
    hadoop001: stopping secondarynamenode
    19/03/16 06:43:37 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    
    [hadoop@hadoop001 tmp]$ jps
    14581 Jps
    13530 DataNode
    [hadoop@hadoop001 tmp]$  #删除了PID文件，发现无法关闭进程













