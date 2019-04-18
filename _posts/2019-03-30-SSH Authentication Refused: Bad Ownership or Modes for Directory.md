---
layout:     post
title:      "ssh信任关系建立后任需要输入密码"
date:       2019-03-30 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - linux
---


### 问题描述
出现这个错误的环境描述，重启hadoop伪分布式的时候，出现了每次启动一个进程都需要输入密码，通过第一印象是配置ssh相互信任关系有问题，但是，我以前的
ssh相互信任是可以正常工作的。检查权限等问题，看/.ssh/authorized_keys 的属性，.ssh目录的权限。经过检查，发现这些都是正常的。
没有找到根本原因，直接把.ssh/删除，重新配置信任关系，还是不行。

### 排查问题

**实时查看/var/log/secure 文件内容**

    [root@hadoop001 ~]# tail -F /var/log/secure
    Mar 30 10:15:10 hadoop001 sshd[3248]: pam_unix(sshd:session): session opened for user root by (uid=0)
    Mar 30 10:15:22 hadoop001 sshd[3272]: Accepted password for root from 192.168.100.109 port 58716 ssh2
    Mar 30 10:15:22 hadoop001 sshd[3272]: pam_unix(sshd:session): session opened for user root by (uid=0)
    Mar 30 10:15:22 hadoop001 sshd[3274]: Accepted password for root from 192.168.100.109 port 58718 ssh2
    Mar 30 10:15:22 hadoop001 sshd[3274]: pam_unix(sshd:session): session opened for user root by (uid=0)
    Mar 30 10:15:32 hadoop001 su: pam_unix(su-l:session): session opened for user hadoop by root(uid=0)
    Mar 30 10:15:37 hadoop001 su: pam_unix(su-l:session): session opened for user hadoop by root(uid=0)
    Mar 30 10:15:44 hadoop001 su: pam_unix(su-l:session): session opened for user hadoop by root(uid=0)
    Mar 30 10:16:01 hadoop001 sshd[3493]: Authentication refused: bad ownership or modes for directory /home/hadoop
    Mar 30 10:17:10 hadoop001 su: pam_unix(su-l:session): session closed for user hadoop
    Mar 30 10:17:52 hadoop001 sshd[3493]: Failed password for hadoop from 192.168.100.111 port 51319 ssh2
    Mar 30 10:17:53 hadoop001 sshd[3493]: Failed password for hadoop from 192.168.100.111 port 51319 ssh2
    Mar 30 10:17:53 hadoop001 sshd[3494]: Connection closed by 192.168.100.111
    Mar 30 10:17:53 hadoop001 sshd[3529]: Authentication refused: bad ownership or modes for directory /home/hadoop
    Mar 30 10:17:54 hadoop001 sshd[3529]: Failed password for hadoop from 192.168.100.111 port 51320 ssh2
    Mar 30 10:17:54 hadoop001 sshd[3529]: Failed password for hadoop from 192.168.100.111 port 51320 ssh2
    Mar 30 10:17:54 hadoop001 sshd[3530]: Connection closed by 192.168.100.111
    Mar 30 10:17:56 hadoop001 sshd[3590]: Authentication refused: bad ownership or modes for directory /home/hadoop
    Mar 30 10:17:57 hadoop001 sshd[3590]: Failed password for hadoop from 127.0.0.1 port 48546 ssh2
    Mar 30 10:17:58 hadoop001 sshd[3590]: Failed password for hadoop from 127.0.0.1 port 48546 ssh2
    Mar 30 10:17:58 hadoop001 sshd[3591]: Connection closed by 127.0.0.1
    Mar 30 10:19:59 hadoop001 sshd[3672]: Authentication refused: bad ownership or modes for directory /home/hadoop
    Mar 30 10:20:44 hadoop001 sshd[3673]: Connection closed by ::1

**找到问题的原因：**

    Mar 30 10:16:01 hadoop001 sshd[3493]: Authentication refused: bad ownership or modes for directory /home/hadoop

说明/home/hadoop权限有问题，但是检查过了
/.ssh/authorized_keys文件的属性，以及.ssh文件属性   是不是权限过大。.ssh目录的权限必须是700，同时/.ssh/authorized_keys的权限必须设置成600：

**执行下面命令**

    chmod g-w /home/hadoop
    
### 验证

    [hadoop@hadoop001 ~]$ start-all.sh 
    This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
    Starting namenodes on [hadoop001]
    hadoop001: starting namenode, logging to /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/logs/hadoop-hadoop-namenode-hadoop001.out
    hadoop001: starting datanode, logging to /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/logs/hadoop-hadoop-datanode-hadoop001.out
    Starting secondary namenodes [0.0.0.0]
    0.0.0.0: starting secondarynamenode, logging to /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/logs/hadoop-hadoop-secondarynamenode-hadoop001.out
    starting yarn daemons
    starting resourcemanager, logging to /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/logs/yarn-hadoop-resourcemanager-hadoop001.out
    hadoop001: starting nodemanager, logging to /home/hadoop/app/hadoop-2.6.0-cdh5.7.0/logs/yarn-hadoop-nodemanager-hadoop001.out
    [hadoop@hadoop001 ~]$ jps
    5220 NameNode
    5524 SecondaryNameNode
    5351 DataNode
    5898 Jps
    5789 NodeManager
    5678 ResourceManager
    [hadoop@hadoop001 ~]$ 

集群可以正常启动了。不建议修改修改/etc/ssh/sshd_config文件,  把密码认证关闭, 将认证改为 passwordAuthentication no  

重启下sshd。

service sshd restart;


**参考博文**
[参考博文](https://www.daveperrett.com/articles/2010/09/14/ssh-authentication-refused/)





