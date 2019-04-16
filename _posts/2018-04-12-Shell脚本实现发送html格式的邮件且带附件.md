---
layout:     post
title:      "Shell脚本实现发送html格式的邮件且带附件"
date:       2018-04-12 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - linux,shell
---

### shell 实现发送带附件的邮件

#### 邮箱开启授权认证
下图为QQ邮箱开启第三方接收邮件的授权认证

![QQ邮箱授权](/img/shell1.png)

其他邮箱做同样的操作

#### 启动postfix
    #sendmial
    service sendmail stop
    chkconfig sendmail off
    
    #postfix
    service postfix start
    chkconfig postfix on
    
**如果postfix start失败**

    [root@hadoop001 ~]# postfix check
    postfix: error while loading shared libraries: libmysqlclient.so.16: cannot open shared object file: No such file or directory
    [root@hadoop001 ~]# rpm -qa|grep mysql
    [root@hadoop001 ~]# yum install mysql-libs

#### 创建认证

    [root@hadoop001 ~]#mkdir -p /root/.certs/
    [root@hadoop001 ~]#echo -n | openssl s_client -connect smtp.qq.com:465 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > ~/.certs/qq.crt
    [root@hadoop001 ~]#certutil -A -n "GeoTrust SSL CA" -t "C,," -d ~/.certs -i ~/.certs/qq.crt
    [root@hadoop001 ~]#certutil -A -n "GeoTrust Global CA" -t "C,," -d ~/.certs -i ~/.certs/qq.crt
    [root@hadoop001 ~]#certutil -L -d /root/.certs
    [root@hadoop001 ~]#cd /root/.certs
    [root@hadoop001 ~]#certutil -A -n "GeoTrust SSL CA - G3" -t "Pu,Pu,Pu"  -d ./ -i qq.crt

**同理，其他邮箱的认证，只需要把qq换成其他邮箱即可，比如163邮箱**

#### 配置mail.rc

    [root@hadoop001 ~]#vi /etc/mail.rc
    
    set from=2391554474@qq.com
    set smtp=smtp.qq.com
    set smtp-auth-user=2391554474
    #授权码
    set smtp-auth-password=liqftivgxxaidihe
    set smtp-auth=login
    set smtp-use-starttls
    set ssl-verify=ignore
    set nss-config-dir=/root/.certs
    

#### 测试
    echo  "hello word" | mail -s "title"  2391554474@qq.com

#### 发邮件不带附件

    [root@hadoop001 shell]# cat mail_noattachment.sh 
    #!/bin/bash 
    
    JOB_NAME="TEST"
    FROM_EMAIL="2391554474@qq.com"
    TO_EMAIL="2391554474@qq.com"
    
    RUNNINGNUM=1
    
    echo -e "`date "+%Y-%m-%d %H:%M:%S"` : The current running $JOB_NAME job num is $RUNNINGNUM in 192.168.137.201 ......" | mail \
    -r "From: alertAdmin <${FROM_EMAIL}>" \
    -s "Warn: Skip the new $JOB_NAME spark job." ${TO_EMAIL}
    [root@hadoop001 shell]# 

#### 发送带附件的邮件

    [root@hadoop001 shell]# cat order.txt
    10703007267488	 2014-05-10 06:01:12.334+01
    10101043509689	 2014-05-10 07:36:23.342+01
    10101043529689	 2014-05-10 07:45:23.32+01
    10101043549689	 2014-05-10 07:51:23.32+01
    10101043539689	 2014-05-10 07:57:23.342+01

    [root@yws75 shell]# cat mail_attachment.sh 
    #!/bin/bash 
    
    
    FROM_EMAIL="2391554474@qq.com"
    TO_EMAIL="2391554474@qq.com"
    
    LOG=/root/shell/order.log
    
    
    echo -e "`date "+%Y-%m-%d %H:%M:%S"` : Please to check the fail sql attachement." | mailx \
    -r "From: alertAdmin <${FROM_EMAIL}>" \
    -a ${LOG} \
    -s "Critical:DSHS fail sql." ${TO_EMAIL}
    [root@hadoop001 shell]# 

**在QQ邮件中会收到带附件order.txt的附件**

### 发送html表格

需要在头部设置Content-Type: text/html

    [hadoop@hadoop001 shell]$ cat mail_back.html 
    <!DOCTYPE html>
    
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Title</title>
    </head>
    <style type="text/css">
    
            table{
    			border-collapse: collapse;
    		}
    		table,th,td{
    			border:1px,solid,black;
    		}
    
    </style>
    <body>
    <table border ="1">
    	<thead>
    		<tr>
    			<th>DATABASE</th>
    			<th>TABLE</th>
    			<th>COUNT</th>
    			<th>HBASE-SCHEMA</th>
    			<th>TABLE</th>
    			<th>COUNT</th>
    			<th>C_TIME</th>
    		
    		</tr>
    	</thead>
    	<tbody>
    		<tr>
    			<td>OMS</td>
    			<td>OMS_T10</td>
    			<td>10000</td>
    			<td>DW</td>
    			<td>OMS_T10</td>
    			<td>10000</td>
    			<td>20190414000001</td>
    		</tr>
    		<tr bgcolor="#FF0000">
    			<td>OMS</td>
    			<td>OMS_T10</td>
    			<td>30000</td>
    			<td>DW</td>
    			<td>OMS_T10</td>
    			<td>29900</td>
    			<td>20190414000002</td>
    		</tr>
    	</tbody>
    </table>
    </body>
    </html>
   
 #### 发送测试
    [hadoop@hadoop001 shell]$cat mail_back.html | mail -s  "$(echo -e "subject\nContent-Type:text/html")" 2391554474@qq.com
   
 **注意：如果用的是QQ邮箱自带web可能会出现下面这种情况**
 
 ![QQ邮箱问题](/img/shell2.png)
 
 **建议大家使用Foxmail收发邮件**
 
 ![解决问题](/img/shell3.png)

**同时也可以用sendEmail发送邮件**

[参考博文](https://my.oschina.net/u/4005872/blog/3035997)










