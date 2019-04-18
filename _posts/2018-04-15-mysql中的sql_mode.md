---
layout:     post
title:      "mysql中的sql_mode"
date:       2018-04-15 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - mysql
---


### 概述

在 mysql 5.7 后，sql_mode是严格模式，严格遵守sql的语法标准。

#### MySQL5.7中的默认SQL模式

    ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_AUTO_CREATE_USER, 
    NO_ENGINE_SUBSTITUTION
    
MySQL 5.7.5 中默认SQL模式添加了 ONLY_FULL_GROUP_BY和STRICT_TRANS_TABLES

MySQL 5.7.7 中默认SQL模式添加了 NO_AUTO_CREATE_USER

MySQL 5.7.8 中默认SQL模式添加了 ERROR_FOR_DIVISION_BY_ZERO，NO_ZERO_DATE 和 NO_ZERO_IN_DATE

#### 查询sql_mode的方式

    查询全局sql_mode
    SELECT @@GLOBAL.sql_mode;
    查询当前会话sql_mode
    SELECT @@SESSION.sql_mode;
    
### 设置sql_mode

sql_mode 的设置有三种方式，分别是启动命令行设置、配置文件设置、运行时设置，多个模式之间使用逗号隔开

### 启动命令行设置

服务启动时，在命令行使用 --sql-mode =“modes” （设置sql_mode） 或 --sql-mode =“” （清空sql_mode）

### 配置文件设置
在Unix操作系统下配置文件 my.cnf 设置 sql-mode =“modes” （设置sql_mode） 或 sql-mode =“”（清空sql_mode）

在Windows系统下配置文件 my.ini 中设置 sql-mode =“modes” 或 sql-mode =“”（清空sql_mode）

### 运行时设置
要在运行时更改SQL模式，使用SET语句设置全局或会话sql_mode系统变量：

    SET GLOBAL sql_mode = 'modes';
    SET SESSION sql_mode = 'modes';
    
### sql_mode常用值
此处只给出了部分值，并非全部。

    ONLY_FULL_GROUP_BY
    
     对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中。
    
    NO_AUTO_VALUE_ON_ZERO
    
     该值影响自增长列的插入。默认设置下，插入0或NULL代表生成下一个自增长值。如果用户希望插入的值为0，该列又是自增长的，那么这个选项就有用了。
    
    STRICT_TRANS_TABLES
    
     在该模式下，如果一个值不能插入到一个事物表中，则中断当前的操作，对非事物表不做限制
    
    NO_ZERO_IN_DATE
    
     在严格模式下，不允许日期和月份为零
    
    NO_ZERO_DATE
    
     设置该值，mysql数据库不允许插入零日期，插入零日期会抛出错误而不是警告。
    
    ERROR_FOR_DIVISION_BY_ZERO
    
     在INSERT或UPDATE过程中，如果数据被零除，则产生错误而非警告。如 果未给出该模式，那么数据被零除时MySQL返回NULL
    
    NO_AUTO_CREATE_USER
    
    禁止GRANT创建密码为空的用户
    
    NO_ENGINE_SUBSTITUTION
    
     如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常
    
    PIPES_AS_CONCAT
    
     将"||"视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似
    
    ANSI_QUOTES
    
    启用ANSI_QUOTES后，不能用双引号来引用字符串，因为它被解释为识别符
        
    HIGH_NOT_PRECEDENCE
    
        NOT运算符的优先级使得诸如 NOT a BETWEEN b AND c 之类的表达式被解析为NOT（a BETWEEN b AND c）
        
        mysql> SET sql_mode = '';
        mysql> SELECT NOT 1 BETWEEN -5 AND 5;
            -> 0
        mysql> SET sql_mode = 'HIGH_NOT_PRECEDENCE';
        mysql> SELECT NOT 1 BETWEEN -5 AND 5;
            -> 1
    
    IGNORE_SPACE
        允许函数名和括号【（】之间的空格。这会导致内置函数名被视为保留字。
        IGNORE_SPACE SQL模式适用于内置函数，而不适用于用户定义的函数或存储函数。 无论是否启用IGNORE_SPACE，始终允许在UDF或存储的函数名后面包含空格。
        
        mysql> CREATE TABLE count (i INT);
        ERROR 1064 (42000): You have an error in your SQL syntax
        
        mysql> CREATE TABLE `count` (i INT);
        Query OK, 0 rows affected (0.00 sec)
        
        
### 测试





    root@db 08:18:  [student]> SET FOREIGN_KEY_CHECKS=0;
    Query OK, 0 rows affected (0.00 sec)
    
    root@db 08:18:  [student]> 
    root@db 08:18:  [student]> DROP TABLE IF EXISTS `test`;
    
    INSERT INTO `test` VALUES ('3', 'c');
    INSERT INTO `test` VALUES ('2', 'a');Query OK, 0 rows affected, 1 warning (0.02 sec)
    
    root@db 08:18:  [student]> CREATE TABLE `test` (
        ->   `id` int(1) DEFAULT NULL,
        ->   `name` varchar(255) DEFAULT NULL
        -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
    Query OK, 0 rows affected (0.09 sec)
    
    root@db 08:18:  [student]> 
    root@db 08:18:  [student]> 
    root@db 08:18:  [student]> INSERT INTO `test` VALUES ('1', 'a');
    Query OK, 1 row affected (0.01 sec)
    
    root@db 08:18:  [student]> INSERT INTO `test` VALUES ('2', 'b');
    Query OK, 1 row affected (0.00 sec)
    
    root@db 08:18:  [student]> INSERT INTO `test` VALUES ('3', 'c');
    Query OK, 1 row affected (0.01 sec)
    
    root@db 08:18:  [student]> INSERT INTO `test` VALUES ('2', 'a');
    Query OK, 1 row affected (0.01 sec)
    
    
    root@db 08:19:  [student]> select id,name from group by name;
    ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'group by name' at line 1
    
    
    
    
    root@db 10:19:  [student]> show variables like '%sql_mode%'\G;
    *************************** 1. row ***************************
    Variable_name: sql_mode
            Value: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
    1 row in set (0.00 sec)
    
    ERROR: 
    No query specified
    
    
    root@db 10:51:  [student]> set session sql_mode='';
    Query OK, 0 rows affected (0.00 sec)
    
    root@db 10:51:  [student]> 
    root@db 10:51:  [student]> 
    root@db 10:51:  [student]> select @@sql_mode;
    +------------+
    | @@sql_mode |
    +------------+
    |            |
    +------------+
    1 row in set (0.00 sec)
    
    root@db 10:52:  [student]> select id,name from test group by name;
    +------+------+
    | id   | name |
    +------+------+
    |    1 | a    |
    |    2 | b    |
    |    3 | c    |
    +------+------+
    3 rows in set (0.00 sec)
    
**注意：在当前会话设置了sql_mode为null，即可以实现一些非标准的sql语法。**    




    
    
    
    
    
    
    
    
    
    
    
    
