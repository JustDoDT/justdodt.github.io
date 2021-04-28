---
layout:     post
title:      "HBase Shell 从入门到精通"
date:       2021-04-27 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - HBase
---

## HBase shell 使用

### 1. 概述

hbase shell 在日常用得比较多，里面有不少常用的命令，进入hbase的shell客户端。**注意**：本文是基于Apache HBase 2.2.3。

~~~shell
[root@cdp01 ~]# hbase shell
HBase Shell
Use "help" to get list of supported commands.
Use "exit" to quit this interactive shell.
For Reference, please visit: http://hbase.apache.org/2.0/book.html#shell
Version 2.2.3.7.1.5.0-257, rUnknown, Thu Nov 26 15:29:39 UTC 2020
Took 0.0031 seconds                                                                                        
hbase:001:0> 
~~~

查看系统所支持的所有命令  help

~~~shell
hbase:001:0> help
HBase Shell, version 2.2.3.7.1.5.0-257, rUnknown, Thu Nov 26 15:29:39 UTC 2020
Type 'help "COMMAND"', (e.g. 'help "get"' -- the quotes are necessary) for help on a specific command.
Commands are grouped. Type 'help "COMMAND_GROUP"', (e.g. 'help "general"') for help on a command group.

COMMAND GROUPS:
  Group name: general
  Commands: processlist, status, table_help, version, whoami

  Group name: ddl
  Commands: alter, alter_async, alter_status, clone_table_schema, create, describe, disable, disable_all, drop, drop_all, enable, enable_all, exists, get_table, is_disabled, is_enabled, list, list_regions, locate_region, show_filters

  Group name: namespace
  Commands: alter_namespace, create_namespace, describe_namespace, drop_namespace, list_namespace, list_namespace_tables

  Group name: dml
  Commands: append, count, delete, deleteall, get, get_counter, get_splits, incr, put, scan, truncate, truncate_preserve

  Group name: tools
  Commands: assign, balance_switch, balancer, balancer_enabled, catalogjanitor_enabled, catalogjanitor_run, catalogjanitor_switch, cleaner_chore_enabled, cleaner_chore_run, cleaner_chore_switch, clear_block_cache, clear_compaction_queues, clear_deadservers, close_region, compact, compact_rs, compaction_state, compaction_switch, decommission_regionservers, flush, hbck_chore_run, is_in_maintenance_mode, list_deadservers, list_decommissioned_regionservers, major_compact, merge_region, move, normalize, normalizer_enabled, normalizer_switch, recommission_regionserver, regioninfo, rit, split, splitormerge_enabled, splitormerge_switch, stop_master, stop_regionserver, trace, unassign, wal_roll, zk_dump

  Group name: replication
  Commands: add_peer, append_peer_exclude_namespaces, append_peer_exclude_tableCFs, append_peer_namespaces, append_peer_tableCFs, disable_peer, disable_table_replication, enable_peer, enable_table_replication, get_peer_config, list_peer_configs, list_peers, list_replicated_tables, remove_peer, remove_peer_exclude_namespaces, remove_peer_exclude_tableCFs, remove_peer_namespaces, remove_peer_tableCFs, set_peer_bandwidth, set_peer_exclude_namespaces, set_peer_exclude_tableCFs, set_peer_namespaces, set_peer_replicate_all, set_peer_serial, set_peer_tableCFs, show_peer_tableCFs, update_peer_config

  Group name: snapshots
  Commands: clone_snapshot, delete_all_snapshot, delete_snapshot, delete_table_snapshots, list_snapshots, list_table_snapshots, restore_snapshot, snapshot

  Group name: configuration
  Commands: update_all_config, update_config

  Group name: quotas
  Commands: disable_exceed_throttle_quota, disable_rpc_throttle, enable_exceed_throttle_quota, enable_rpc_throttle, list_quota_snapshots, list_quota_table_sizes, list_quotas, list_snapshot_sizes, set_quota

  Group name: security
  Commands: grant, list_security_capabilities, revoke, user_permission

  Group name: procedures
  Commands: list_locks, list_procedures

  Group name: visibility labels
  Commands: add_labels, clear_auths, get_auths, list_labels, set_auths, set_visibility

  Group name: rsgroup
  Commands: add_rsgroup, balance_rsgroup, get_rsgroup, get_server_rsgroup, get_table_rsgroup, list_rsgroups, move_namespaces_rsgroup, move_servers_namespaces_rsgroup, move_servers_rsgroup, move_servers_tables_rsgroup, move_tables_rsgroup, remove_rsgroup, remove_servers_rsgroup

SHELL USAGE:
Quote all names in HBase Shell such as table and column names.  Commas delimit
command parameters.  Type <RETURN> after entering a command to run it.
Dictionaries of configuration used in the creation and alteration of tables are
Ruby Hashes. They look like this:

  {'key1' => 'value1', 'key2' => 'value2', ...}

and are opened and closed with curley-braces.  Key/values are delimited by the
'=>' character combination.  Usually keys are predefined constants such as
NAME, VERSIONS, COMPRESSION, etc.  Constants do not need to be quoted.  Type
'Object.constants' to see a (messy) list of all constants in the environment.

If you are using binary keys or values and need to enter them in the shell, use
double-quote'd hexadecimal representation. For example:

  hbase> get 't1', "key\x03\x3f\xcd"
  hbase> get 't1', "key\003\023\011"
  hbase> put 't1', "test\xef\xff", 'f1:', "\x01\x33\x40"

The HBase shell is the (J)Ruby IRB with the above HBase-specific commands added.
For more on the HBase Shell, see http://hbase.apache.org/book.html
hbase:002:0> 
~~~

具体查看某个命令的详细使用说明  help ‘command’

~~~shell
hbase:003:0> help 'processlist'
Show regionserver task list.

  hbase> processlist
  hbase> processlist 'all'
  hbase> processlist 'general'
  hbase> processlist 'handler'
  hbase> processlist 'rpc'
  hbase> processlist 'operation'
  hbase> processlist 'all','host187.example.com'
  hbase> processlist 'all','host187.example.com,16020'
  hbase> processlist 'all','host187.example.com,16020,1289493121758'
~~~

### 2. 通用命令

#### 2.1 status

查看集群状态，有三种可选的参数 simple、summary、detailed、replication。默认为 summary。

~~~shell
hbase:004:0> help 'status'
Show cluster status. Can be 'summary', 'simple', 'detailed', or 'replication'. The
default is 'summary'. Examples:

  hbase> status
  hbase> status 'simple'
  hbase> status 'summary'
  hbase> status 'detailed'
  hbase> status 'replication'
  hbase> status 'replication', 'source'
  hbase> status 'replication', 'sink'
~~~

**示例：**

~~~shell
hbase:009:0> status 'summary'
1 active master, 0 backup masters, 3 servers, 0 dead, 2.3333 average load
Took 0.0263 seconds             


hbase:007:0> status 'replication', 'sink'
version 2.2.3.7.1.5.0-257
3 live servers
    cdp01:
       SINK  : AgeOfLastAppliedOp=0, TimeStampsOfLastAppliedOp=Thu Apr 22 16:55:38 CST 2021
    cdp02:
       SINK  : AgeOfLastAppliedOp=0, TimeStampsOfLastAppliedOp=Thu Apr 22 16:55:48 CST 2021
    cdp03:
       SINK  : AgeOfLastAppliedOp=0, TimeStampsOfLastAppliedOp=Thu Apr 22 16:55:46 CST 2021
Took 0.0264 seconds                                                                                        
=> #<Java::JavaUtil::Collections::UnmodifiableSet:0x3321291a>


hbase:010:0> status 'simple'
active master:  cdp01:16000 1619081725523
0 backup masters
3 live servers
    cdp01:16020 1619081724381
        requestsPerSecond=0.0, numberOfOnlineRegions=3, usedHeapMB=44, maxHeapMB=48, numberOfStores=12, numberOfStorefiles=6, storefileUncompressedSizeMB=0, storefileSizeMB=0, memstoreSizeMB=0, storefileIndexSizeKB=0, readRequestsCount=1096, filteredReadRequestsCount=0, writeRequestsCount=0, rootIndexSizeKB=0, totalStaticIndexSizeKB=0, totalStaticBloomSizeKB=0, totalCompactingKVs=0, currentCompactedKVs=0, compactionProgressPct=NaN, coprocessors=[RangerAuthorizationCoprocessor, SecureBulkLoadEndpoint, TokenProvider]
    cdp02:16020 1619081727904
        requestsPerSecond=0.0, numberOfOnlineRegions=2, usedHeapMB=41, maxHeapMB=48, numberOfStores=5, numberOfStorefiles=7, storefileUncompressedSizeMB=0, storefileSizeMB=0, memstoreSizeMB=0, storefileIndexSizeKB=0, readRequestsCount=81161, filteredReadRequestsCount=2589, writeRequestsCount=48, rootIndexSizeKB=0, totalStaticIndexSizeKB=0, totalStaticBloomSizeKB=0, totalCompactingKVs=127, currentCompactedKVs=127, compactionProgressPct=1.0, coprocessors=[MultiRowMutationEndpoint, RangerAuthorizationCoprocessor, SecureBulkLoadEndpoint, TokenProvider]
    cdp03:16020 1619081727753
        requestsPerSecond=0.0, numberOfOnlineRegions=2, usedHeapMB=38, maxHeapMB=48, numberOfStores=4, numberOfStorefiles=3, storefileUncompressedSizeMB=0, storefileSizeMB=0, memstoreSizeMB=0, storefileIndexSizeKB=0, readRequestsCount=9430, filteredReadRequestsCount=181, writeRequestsCount=0, rootIndexSizeKB=0, totalStaticIndexSizeKB=0, totalStaticBloomSizeKB=0, totalCompactingKVs=0, currentCompactedKVs=0, compactionProgressPct=NaN, coprocessors=[RangerAuthorizationCoprocessor, SecureBulkLoadEndpoint, TokenProvider]
0 dead servers
Aggregate load: 0, regions: 7
Took 0.0170 seconds                                                                                        
hbase:011:0> 
~~~

#### 2.2 version

查看当前 HBase 版本

~~~shell
hbase:013:0> version
2.2.3.7.1.5.0-257, rUnknown, Thu Nov 26 15:29:39 UTC 2020
Took 0.0004 seconds                                       
~~~

#### 2.3 whoami

查看当前用户

~~~shell
hbase:018:0> whoami
testuser@MACRO.COM (auth:KERBEROS)
    groups: testuser
Took 0.0445 seconds    
~~~

#### 2.4 table_help

用于输出关于表操作的帮助信息

~~~shell
hbase:020:0> help 'table_help'
Help for table-reference commands.

You can either create a table via 'create' and then manipulate the table via commands like 'put', 'get', etc.
See the standard help information for how to use each of these commands.

However, as of 0.96, you can also get a reference to a table, on which you can invoke commands.
For instance, you can get create a table and keep around a reference to it via:

   hbase> t = create 't', 'cf'

Or, if you have already created the table, you can get a reference to it:

   hbase> t = get_table 't'

You can do things like call 'put' on the table:

  hbase> t.put 'r', 'cf:q', 'v'

which puts a row 'r' with column family 'cf', qualifier 'q' and value 'v' into table t.

To read the data out, you can scan the table:

  hbase> t.scan

which will read all the rows in table 't'.

Essentially, any command that takes a table name can also be done via table reference.
Other commands include things like: get, delete, deleteall,
get_all_columns, get_counter, count, incr. These functions, along with
the standard JRuby object methods are also available via tab completion.

For more information on how to use each of these commands, you can also just type:

   hbase> t.help 'scan'

which will output more information on how to use that command.

You can also do general admin actions directly on a table; things like enable, disable,
flush and drop just by typing:

   hbase> t.enable
   hbase> t.flush
   hbase> t.disable
   hbase> t.drop

Note that after dropping a table, your reference to it becomes useless and further usage
is undefined (and not recommended).
~~~

####2.5 processlist

显示 regionserver 任务列表

~~~shell
hbase:082:0> help 'processlist'
Show regionserver task list.

  hbase> processlist
  hbase> processlist 'all'
  hbase> processlist 'general'
  hbase> processlist 'handler'
  hbase> processlist 'rpc'
  hbase> processlist 'operation'
  hbase> processlist 'all','host187.example.com'
  hbase> processlist 'all','host187.example.com,16020'
  hbase> processlist 'all','host187.example.com,16020,1289493121758'
~~~

**示例：**

~~~shell
hbase:086:0> processlist 'handler'
29 tasks as of: 2021-04-26 15:19:01
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| Host            | Start Time          | State    | Description                      | Status                               |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:55 | WAITING  | RpcServer.default.FPBQ.Fifo.h... | Waiting for a call (since 339786 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339798 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:46 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339795 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339798 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339798 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339798 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:42 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339799 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:42 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339799 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:38 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339803 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339798 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp01           | 2021-04-22 16:55:43 | WAITING  | RpcServer.priority.RWQ.Fifo.w... | Waiting for a call (since 339798 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:01:04 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339477 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:01:04 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339477 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:01:04 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339477 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:01:04 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339477 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:01:04 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339477 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:01:04 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339477 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 16:55:55 | WAITING  | RpcServer.default.FPBQ.Fifo.h... | Waiting for a call (since 339786 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:00:53 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339488 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:00:52 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339489 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:00:45 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339496 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp02           | 2021-04-22 17:00:47 | WAITING  | RpcServer.priority.RWQ.Fifo.w... | Waiting for a call (since 339494 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 16:55:55 | WAITING  | RpcServer.default.FPBQ.Fifo.h... | Waiting for a call (since 339786 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-25 21:20:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 64698 s... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-25 12:30:44 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 96497 s... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 17:00:51 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339490 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 17:00:48 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339493 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 16:55:55 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339786 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 16:55:55 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339786 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
Took 0.5432 seconds      


hbase:094:0> processlist 'all','cdp03,16020'
7 tasks as of: 2021-04-26 15:23:32
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| Host            | Start Time          | State    | Description                      | Status                               |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 16:55:55 | WAITING  | RpcServer.default.FPBQ.Fifo.h... | Waiting for a call (since 340057 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-25 21:20:43 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 64969 s... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-25 12:30:44 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 96768 s... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 17:00:51 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339761 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 17:00:48 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 339764 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 16:55:55 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 340057 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
| cdp03           | 2021-04-22 16:55:55 | WAITING  | RpcServer.priority.RWQ.Fifo.r... | Waiting for a call (since 340057 ... |
+-----------------+---------------------+----------+----------------------------------+--------------------------------------+
Took 0.0405 seconds                                            

~~~



### 3. 表操作

#### 3.1 list

列出所有表名

~~~shell
hbase:021:0> help 'list'
List all user tables in hbase. Optional regular expression parameter could
be used to filter the output. Examples:

  hbase> list
  hbase> list 'abc.*'
  hbase> list 'ns:abc.*'
  hbase> list 'ns:.*'
  
  
hbase:023:0> list
TABLE                                                                                                      
atlas_janus                                                                                                
test                                                                                                       
user1                                                                                                      
user_info                                                                                                  
4 row(s)
Took 0.0463 seconds                                                                                        
=> ["atlas_janus", "test", "user1", "user_info"]


hbase:026:0> list 'user.*'
TABLE                                                                                                      
user1                                                                                                      
user_info                                                                                                  
2 row(s)
Took 0.0058 seconds                                                                                        
=> ["user1", "user_info"]  
  
~~~

#### 3.2 alter

更改表或者列族定义。如果你传入一个新的列族名，则意味着创建一个新的列族。

（1）建立、修改列族

如果传入新的列族名，可以新建列族；如果传入已经存在的列族名，可以修改列族属性。列族属性有：

~~~shell
NAME
VERSIONS
EVICT_BLOCKS_ON_CLOSE
NEW_VERSION_BEHAVIOR
KEEP_DELETED_CELLS
CACHE_DATA_ON_WRITE
DATA_BLOCK_ENCODING 
TTL  
MIN_VERSIONS  
REPLICATION_SCOPE
BLOOMFILTER
CACHE_INDEX_ON_WRITE  
IN_MEMORY
CACHE_BLOOMS_ON_WRITE  
PREFETCH_BLOCKS_ON_OPEN
COMPRESSION 
BLOCKCACHE  
BLOCKSIZE 
~~~

**格式：**

alter ‘表名’，NAME => ‘列族名’ , 属性名1 => 属性值1，属性名2 => 属性值2，….

**示例**

~~~shell
hbase:053:0> alter 'user1',NAME => 'cf1',VERSIONS =>2,TTL => 2000,NEW_VERSION_BEHAVIOR => true
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 2.8072 seconds          
~~~

（2）建立、修改多个列族

**格式：**

alter ‘表名’，{NAME => ‘列族名1’，属性名1 => 属性值1，属性名2 => 属性值2，……}，{NAME => ‘列族名2’，属性名1 => 属性值1，属性名2 => 属性值2，……}

**示例：**

~~~shell
hbase:064:0> alter 'test',{NAME => 'cf',EVICT_BLOCKS_ON_CLOSE => true,CACHE_DATA_ON_WRITE => true,TTL => 20000},{NAME => 'cf2',EVICT_BLOCKS_ON_CLOSE => true,CACHE_DATA_ON_WRITE => true,VERSIONS => 3}
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 3.3519 seconds
~~~

（3）删除列族

**格式：**

alter ‘表名’，‘delete’ => ‘列族名’

~~~shell
hbase:066:0> alter 'test','delete' => 'cf2'
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 3.1696 seconds      
~~~

（4）修改表级别属性

允许的属性名必须是属于表级别的属性。表级别的属性有：

- MAX_FILESIZE
- READONLY
- MEMSTORE_FLUSHSIZE
- DURABILITY
- REGION_REPLICATION
- NORMALIZATION_ENABLED
- PRIORITY

**格式：**

alter ‘表名’，属性名1 => 属性值1，属性名2 => 属性值2，……

**示例：**

~~~shell
hbase:096:0> alter 'test',MAX_FILESIZE => '12345678'
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 2.8814 seconds          
~~~

（5）设置表配置

一般强开下，我们都会把表或者列族的配置属性设置在 hbase-site.xml 文件里面。现在，alter 命令给你了一个专属修改这个表或者列族配置属性值的机会。比如，我们在 hbase-site.xml 文件配置的 hbase.hstore.blockingStoreFiles 是 10，我们可以将该列族的 hbase.hstore.blockingStoreFiles 修改为 20，而不影响到别的表。

**格式：**

- alter ‘表名’，CONFIGURATION => {‘配置名’ => ‘配置值’}
- alter ‘表名’ ，{NAME => ‘列族名’，CONFIGURATION => {‘配置名’ => ‘配置值’}}

**示例：**

~~~shell
hbase:102:0> alter 'test',CONFIGURATION => {'hbase.hstore.blockingStoreFiles' => '20'}
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 3.3630 seconds    


hbase:107:0> alter 'user1',{NAME => 'cf1',CONFIGURATION => {'hbase.hstore.blockingStoreFiles' => '20'}}
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 2.4105 seconds       
~~~

（6）删除表级属性：

**格式：**

alter ‘表名’，METHOD => ‘delete’ , NAME => ‘属性名’

**示例：**

~~~shell
hbase:113:0> alter 'user1',METHOD => 'delete',NAME => 'data1'
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 2.7178 seconds      
~~~

（7）同时执行多个命令

您还可以把前面说的那些命令都放到一条命令里面去执行。

**格式：**

alter ‘表名’ , 命令1，命令2，命令3

**示例：**

~~~shell
hbase:119:0> alter 'user1',{NAME => 'cf2',VERSIONS => 4},{MAX_FILESIZE => '12345678'},{METHOD => 'delete',NAME => 'cf1'}
Updating all regions with the new schema...
1/1 regions updated.
Done.
Took 3.2624 seconds   
~~~

#### 3.3 create

建立新表，建立新表的时候同时可以修改表属性。

**格式：**

create ‘表名’,{NAME => ‘列族名1’，属性名 => ‘属性值’}，{NAME => ‘列族名2’，属性名 => 属性值} , …….

如果你只需要创建列族，而不需要定义列族属性，那么可以采用以下快捷写法：

create ‘表名’，‘列族名1’，‘列族名2’，……

**示例：**

~~~shell
hbase:123:0> create 'table2',{NAME => 'cf1',VERSIONS => 5},{NAME => 'cf2'},{NAME => 'cf3',CONFIGURATION => {'hbase.hstore.blockingStoreFiles' => '20'}}
Created table table2
Took 1.3662 seconds                                                                                        
=> Hbase::Table - table2
~~~

#### 3.4 alter_status

查看表的各个 Region 的更新状况，这条命令在异步更新表的时候，用来查看更改命令执行的情况，判断该命令是否执行完毕。

**格式：**

alter_status

**示例：**

~~~shell
hbase:125:0> alter_status 'table2'
1/1 regions updated.
Done.
Took 1.0256 seconds    
~~~

#### 3.5 alter_async

异步更新表。使用这个命令你不需要等待表的全部 Region 更新完后才返回。记得配合 alter_status 来检查异步表更改命令的执行进度。

**格式：**

alter_async ‘表名’ , 参数列表

**示例：**

~~~shell
hbase:127:0> alter_async 'table2',NAME => 'cf1',VERSIONS => 10
Took 2.0202 seconds                                                                                        

hbase:130:0> alter_status 'table2'
1/1 regions updated.
Done.
Took 1.0158 seconds          
~~~

#### 3.6 describe 

此命令和 desc 是一样的效果，都是输出表的描述信息。

**格式：**

describe ‘表名’

~~~shell
hbase:132:0> describe 'test'
Table test is ENABLED                                                                                      
test, {TABLE_ATTRIBUTES => {MAX_FILESIZE => '12345678', METADATA => {'hbase.hstore.blockingStoreFiles' => '
20'}}}                                                                                                     
COLUMN FAMILIES DESCRIPTION                                                                                
{NAME => 'cf', VERSIONS => '5', EVICT_BLOCKS_ON_CLOSE => 'true', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELE
TED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'true', DATA_BLOCK_ENCODING => 'NONE', TTL => '20000 SECONDS (
5 HOURS 33 MINUTES 20 SECONDS)', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE
_INDEX_ON_WRITE => 'false', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN
 => 'false', COMPRESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE => '65536'}                            

1 row(s)

QUOTAS                                                                                                     
0 row(s)
Took 0.0573 seconds       
~~~

#### 3.7 disable

停用指定表

**格式：**

disable ‘表名’

**示例：**

~~~shell
hbase:136:0> disable 'table2'
Took 1.3692 seconds          
~~~

#### 3.8 disable_all 

通过正则表达式来停用多个表

**格式：**

disable_all ‘正则表达式’

**示例：**

~~~shell
hbase:139:0> disable_all 'user.*'
user1                                                                                                      
user_info                                                                                                  

Disable the above 2 tables (y/n)?

Took 2.0429 seconds                
~~~

#### 3.9 is_disabled

检测指定表是否被停用了

**格式：**

is_disabled ‘表名’

**示例：**

~~~shell
hbase:001:0> is_disabled 'table2'
true                                                                                                       
Took 0.7338 seconds                                                                                        
=> 1
~~~

#### 3.10 drop

删除指定表

**格式：**

drop ‘表名’

**示例：**

~~~shell
hbase:002:0> drop 'table2'
Took 1.1845 seconds 
~~~

#### 3.11 drop_all

通过正则表达式来删除多个表

**格式：**

drop_all ‘正则表达式’

**示例：**

~~~shell
hbase:030:0> drop_all 'table.*'
table1                                                                                                     
table2                                                                                                     

Drop the above 2 tables (y/n)?
y
2 tables successfully dropped
Took 3.8940 seconds            
~~~

#### 3.12 enable

启动指定表

**格式：**

enable ‘表名’

**示例：**

~~~shell
hbase:037:0> enable 'test'
Took 1.2528 seconds    
~~~

#### 3.13 enable_all

通过正则表达式来启动指定表

**格式：**

enable_all ‘正则表达式’

**示例：**

~~~shell
user_info                                                                                                  

Enable the above 2 tables (y/n)?
y
2 tables successfully enabled
Took 7.1732 seconds                                        
~~~

#### 3.14 is_enabled

判断指定表是否启用

**格式：**

is_enabled ‘表名’

**示例：**

~~~~shell
hbase:046:0> is_enabled 'user1'
true                                                                                                       
Took 0.0339 seconds                                                                                        
=> true
~~~~

#### 3.15 exists

判断指定表是否存在

**格式：**

exists ‘表名’

**示例：**

~~~shell
hbase:004:0> exists 'atlas_janus'
Table atlas_janus does exist                                                                      
Took 0.0285 seconds                                                                               
=> true
hbase:005:0> exists 'atlas'
Table atlas does not exist                                                                        
Took 0.0071 seconds                                                                               
=> false
~~~

#### 3.16 show_filters

列出所有过滤器

**格式：**

show_filters

**示例：**

~~~shell
hbase:066:0> show_filters
DependentColumnFilter                                                                                      
KeyOnlyFilter                                                                                              
ColumnCountGetFilter                                                                                       
SingleColumnValueFilter                                                                                    
PrefixFilter                                                                                               
SingleColumnValueExcludeFilter                                                                             
FirstKeyOnlyFilter                                                                                         
ColumnRangeFilter                                                                                          
ColumnValueFilter                                                                                          
TimestampsFilter                                                                                           
FamilyFilter                                                                                               
QualifierFilter                                                                                            
ColumnPrefixFilter                                                                                         
RowFilter                                                                                                  
MultipleColumnPrefixFilter                                                                                 
InclusiveStopFilter                                                                                        
PageFilter                                                                                                 
ValueFilter                                                                                                
ColumnPaginationFilter                                                                                     
Took 0.0107 seconds                                                                                        
=> #<Java::JavaUtil::HashMap::KeySet:0x261275ae>
~~~

#### 3.17 get_table

使用该命令，你可以把表名转化成一个对象，在下面的脚本中使用这个对象来操作表，达到面向对象的语法风格。这个命令对表并没有实质性的操作，只是让你的脚本看来更好看，类似一种语法糖。

**格式：**

变量 = get_table ‘表名’

**示例：**

~~~shell
hbase:105:0> t1 = get_table 'test'
Took 0.0005 seconds                                                                                        
=> Hbase::Table - test
hbase:106:0> t1.scan
ROW                         COLUMN+CELL                                                                    
 row1                       column=cf:name, timestamp=1619439806212, value=jack                            
1 row(s)
Took 0.0059 seconds        
~~~

#### 3.18 locate_region

通过这条命令可以知道你所传入的行键（rowkey）对应的行（row）在哪个 Region 里面。

**格式：**

locate_region ‘表名’ , ‘行键’

**示例：**

~~~shell
hbase:108:0> locate_region 'test','row1'
HOST                        REGION                                                                         
 cdp02:16020                {ENCODED => 797a96a1f98775885db2a0456f9aacb3, NAME => 'test,,1619324005040.797a
                            96a1f98775885db2a0456f9aacb3.', STARTKEY => '', ENDKEY => ''}                  
1 row(s)
Took 0.0098 seconds                                                                                        
=> #<Java::OrgApacheHadoopHbase::HRegionLocation:0x4070ace9>
~~~

### 4. 数据操作

#### 4.1 scan

按照行键的字典排序来遍历指定表的数据。遍历所有数据所有列族。

**格式：**

scan ‘表名’

**示例：**

~~~shell
hbase:110:0> scan 'test'
ROW                         COLUMN+CELL                                                                    
 row1                       column=cf:name, timestamp=1619439806212, value=jack                            
1 row(s)
Took 0.3924 seconds    
~~~

（1）指定列

只遍历指定的列，就像我们在关系型数据库中用 select 语句做的事情一样。要注意的是，写列名的时候记得把列族名带上，就像这样 cf1:name

**格式：**

scan ‘表名’，{COLUMNS => [‘列1’，‘列2’，…..]}

**示例：**

~~~shell
hbase:114:0> scan 'test',{COLUMNS => ['cf:name']}
ROW                         COLUMN+CELL                                                                    
 row1                       column=cf:name, timestamp=1619439806212, value=jack                            
1 row(s)
Took 0.0190 seconds     
~~~

（2）指定行键范围

通过传入起始行键（STARTROW）和结束行键（ENDROW）来遍历指定行键范围的记录。强烈建议每次调用 scan 都至少指定起始行键或者结束行键，这会极大的加速遍历速度。

**格式：**

scan ‘表名’ , {STARTROW => ‘起始行键’ ，ENDROW => ‘结束行键’}

**实例：**

~~~shell
hbase:125:0> scan 'user_info',{STARTROW => 'zhangsan_20150701_0007',ENDROW => 'zhangsan_20150701_0008' }
ROW                         COLUMN+CELL                                                                    
 zhangsan_20150701_0007     column=base_info:age, timestamp=1618929528438, value=27                        
 zhangsan_20150701_0007     column=base_info:name, timestamp=1618925605771, value=zhangsan7                
 zhangsan_20150701_0007     column=extra_info:Hobbies, timestamp=1618929922393, value=music                
1 row(s)
Took 0.0169 seconds 



hbase:157:0> scan 'user_info',{STARTROW => 'zhangsan_20150701_0007',COLUMNS => ['base_info:name'],ENDROW =>'zhangsan_20150701_0008' }
ROW                                      COLUMN+CELL                                                                                                         
 zhangsan_20150701_0007                  column=base_info:name, timestamp=1618925605771, value=zhangsan7                                                     
1 row(s)
Took 0.0067 seconds        
~~~

（3）指定最大返回行数量

通过传入最大返回行数量（LIMIT）来控制返回行的数量。类似我们在传统关系型数据库中使用 limit 语句的效果。

**格式：**

scan ‘表名’，{LIMIT => 行数量}

**示例：**

~~~shell
hbase:162:0> scan 'user_info',{STARTROW => 'zhangsan_20150701_0007',LIMIT => 1 }
ROW                       COLUMN+CELL                                                             
 zhangsan_20150701_0007   column=base_info:age, timestamp=1618929528438, value=27                 
 zhangsan_20150701_0007   column=base_info:name, timestamp=1618925605771, value=zhangsan7         
 zhangsan_20150701_0007   column=extra_info:Hobbies, timestamp=1618929922393, value=music         
1 row(s)
Took 0.0137 seconds                                                              
~~~

（4）指定时间戳范围

通过指定时间戳范围（TIMERANGE）来遍历记录，可以使用它来找出单元格的历史版本数据。

**注意：**返回结果包含最小时间戳的记录，但是不包含最大时间戳记录。这是一个左闭右开区间。

**格式：**

scan ‘表名’，{TIMERANGE => [最小时间戳，最大时间戳]}

**示例：**

~~~shell
hbase:170:0> scan 'user_info',{TIMERANGE => [1618925615188,1618925615190]}
ROW                       COLUMN+CELL                                                             
 zhangsan_20150701_0008   column=base_info:name, timestamp=1618925615188, value=zhangsan8         
1 row(s)
Took 0.0137 seconds 
~~~

（5）显示单元格的多个版本值

通过指定版本数（VERSIONS），可以显示单元格的多个版本值。

**格式：**

scan ‘表名’ , {VERSIONS => 版本数}

~~~shell
hbase:019:0> scan 'test',{VERSIONS => 2}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:name, timestamp=1619444732140, value=kobe                     
 row1                     column=cf:name, timestamp=1619439806212, value=jack                     
1 row(s)
Took 0.0074 seconds      
~~~

（6）显示原始单元格记录

在 HBase 中被删除的记录并不会立即从磁盘删除，而是先把打上墓碑标记，然后等待下次 major compaction 的时候再被删除。所谓的原始单元格记录就是连已经被标记为删除但是还未被删除的记录都显示出来。通过添加 RAW 参数来显示原始记录，不过这个参数必须配合 VERSIONS 参数一起使用。RAW 参数不能跟 COLUMNS 参数一起使用。

**格式：**

scan ‘表名’ ， {RAW => true , VERSIONS => 版本数}

**示例：**

~~~shell
hbase:033:0> scan 'test',{RAW => true,VERSIONS => 5}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:name, timestamp=1619445444151, type=Delete                    
 row1                     column=cf:name, timestamp=1619445444151, value=haha                     
 row1                     column=cf:name, timestamp=1619444732140, value=kobe                     
 row1                     column=cf:name, timestamp=1619439806212, value=jack                     
 row1                     column=cf:name, timestamp=1619324233137, value=jack                     
 row2                     column=cf:name, timestamp=1619331007464, value=jack                     
 row3                     column=cf:name, timestamp=1619325568740, value=ted                      
 row3                     column=cf:name, timestamp=1619325558177, value=billy                    
 row4                     column=cf:name, timestamp=1619398112004, value=\x00\x00\x00\x00\x00\x00\
                          x00\x0A                                                                 
 row4                     column=cf:name, timestamp=1619398075689, value=\x00\x00\x00\x00\x00\x00\
                          x00\x14                                                                 
4 row(s)
Took 0.0364 seconds   
~~~

根据上面的实例操作，你可以看见时间戳为 1619445444151 的墓碑标记在记录值为 haha 的记录里，这意味着 haha 这个记录已经被标记删除了。

（7）指定过滤器

通过使用 FILTER 参数来指定要使用的过滤器。

**格式：**

scan ‘表名’ ，{FILTER => ‘过滤器’}

~~~shell
hbase:049:0> scan 'test',{FILTER => "PrefixFilter('row1')"}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:city, timestamp=1619446052644, value=chongqing                
 row1                     column=cf:name, timestamp=1619445999809, value=kangkang                 
1 row(s)
Took 0.0063 seconds     
~~~

多个过滤器可以用 AND 或者 OR 来连接

~~~shell
hbase:057:0> scan 'test',{FILTER => "PrefixFilter('row1') OR PrefixFilter('row2')"}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:city, timestamp=1619446052644, value=chongqing                
 row1                     column=cf:name, timestamp=1619445999809, value=kangkang                 
 row2                     column=cf:city, timestamp=1619446123905, value=chengdu                  
2 row(s)
Took 0.0118 seconds
~~~

#### 4.2 get

通过行键获取某行记录

**格式：**

get ‘表名’ ， ‘行键’

**示例：**

~~~shell
hbase:066:0> get 'test','row1',{COLUMNS => ['cf:name','cf:city']}
COLUMN                    CELL                                                                    
 cf:city                  timestamp=1619446052644, value=chongqing                                
 cf:name                  timestamp=1619445999809, value=kangkang                                 
1 row(s)
Took 0.0567 seconds
~~~

get 支持 scan 所支持的大部分属性，具体支持的属性如下：

- COLUMNS
- TIMERANGE
- VERSIONS
- FILTER

#### 4.3 count

计算表的行数，简单计算。

**格式：**

count ‘表名’

**示例：**

~~~shell
hbase:067:0> count 'test'
2 row(s)
Took 0.0354 seconds                                                                               
=> 2
~~~

（1）指定计算步长

通过指定 INTERVAL 参数来指定步长。如果你使用不带参数的 count 命令，要等到所有行数都计算完毕才能显示结果；如果指定了 INTERVAL 参数，则 shell 会立即显示当前计算的行数结果和当前所在的行键。

**格式：**

count ‘表名’，INTERVAL => 行数计算步长

**示例：**

~~~shell
hbase:070:0> count 'test',INTERVAL => 1
Current count: 1, row: row1                                                                       
Current count: 2, row: row2                                                                       
2 row(s)
Took 0.0083 seconds                                                                               
=> 2
~~~

（2）指定缓存

通过指定缓存加速计算过程。注意：INTERVAL 和 CACHE 是可以同时使用的。

**格式：**

count ‘表名’ ， CACHE => 缓存条数

**示例：**

~~~shell
hbase:072:0> count 'test',CACHE =>2,INTERVAL =>1
Current count: 1, row: row1                                                                       
Current count: 2, row: row2                                                                       
2 row(s)
Took 0.0236 seconds                                                                               
=> 2
~~~

#### 4.4 delete

删除某个列的数据

**格式：**

- delete ‘表名’ ，‘行键’ ，‘列名’
- delete ‘表名’ ，‘行键’ ，‘列名’ ，时间戳

**示例：**

~~~shell
hbase:073:0> scan 'test',{VERSIONS => 5}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:city, timestamp=1619446052644, value=chongqing                
 row1                     column=cf:name, timestamp=1619445999809, value=kangkang                 
 row1                     column=cf:name, timestamp=1619444732140, value=kobe                     
 row1                     column=cf:name, timestamp=1619439806212, value=jack                     
 row2                     column=cf:city, timestamp=1619446123905, value=chengdu                  
2 row(s)
Took 0.0261 seconds                                                                               
hbase:074:0> delete 'test','row1','cf:name',1619439806212
Took 0.0153 seconds                                                                               
hbase:075:0> scan 'test',{VERSIONS => 5}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:city, timestamp=1619446052644, value=chongqing                
 row1                     column=cf:name, timestamp=1619445999809, value=kangkang                 
 row1                     column=cf:name, timestamp=1619444732140, value=kobe                     
 row2                     column=cf:city, timestamp=1619446123905, value=chengdu                  
2 row(s)
Took 0.0094 seconds
~~~

#### 4.5 deleteall

可以使用 deleteall 删除整行数据，也可以删除单列数据，它就像是 delete 的增强版。

**格式：**

- deleteall ‘表名’，‘行键’
- deleteall ‘表名’ ， ‘行键’ ，‘列名’
- deleteall ‘表名’ ，‘行键’ ，‘列名’ ，时间戳

**示例：**

~~~shell
hbase:078:0> deleteall 'test','row1','cf:city',1619446052644
Took 0.0175 seconds 

hbase:079:0> scan 'test',{RAW=>true,VERSIONS=>5}
ROW                       COLUMN+CELL                                                             
 row1                     column=cf:city, timestamp=1619446052644, type=DeleteColumn              
 row1                     column=cf:city, timestamp=1619446052644, value=chongqing  
~~~

#### 4.6 incr

为计数器单元格的值加 1，如果该单元格不存在，则创建一个计算器单元格。所谓计算器单元格就是一个可以做原子加减计算的特殊单元格。

**格式：**

- incr ‘表名’ ，‘行键’ ，‘列名’
- incr ‘表名’ ， ‘行键’ ， ‘列名’ ，加减值

**示例：**

~~~shell
hbase:103:0> incr 'test','row3','cf:count', 3
COUNTER VALUE = 3
Took 0.0172 seconds                                                                               

hbase:108:0> incr 'test','row3','cf:count', -1
COUNTER VALUE = 2
Took 0.0202 seconds               
~~~

#### 4.7 put

put 操作在新增记录的同时还可以为记录设置属性。

**格式：**

- put ‘表名’ , ‘行键’，‘列名’，‘值’
- put ‘表名’ , ‘行键’，‘列名’，‘值’，时间戳
- put ‘表名’ , ‘行键’，‘列名’，‘值’，{‘属性名’ => ‘属性值’}
- put ‘表名’ , ‘行键’，‘列名’，‘值’，时间戳，{‘属性名’ => ‘属性值’}
- put ‘表名’ , ‘行键’，‘列名’，‘值’，{ ATTRIBUTES =>  { ‘属性名’ => ‘属性值’}}
- put ‘表名’ , ‘行键’，‘列名’，‘值’，时间戳，{ ATTRIBUTES =>  { ‘属性名’ => ‘属性值’}}
- put ‘表名’ , ‘行键’，‘列名’，‘值’，时间戳，{ VISIBILITY =>  ‘PRIVATE|SECRET’}

**示例：**

~~~shell
hbase:112:0> put 'test','row3','cf:city','chongqing',{TTL => 3000000}
Took 0.1046 seconds
~~~

#### 4.8 append

给某个单元格的值拼接上新的值。原本我们要给单元格的值拼接新值，需要先 get 出这个单元格的值，拼接上新值后再 put 回去。append 这个操作简化了这两步操作为一步操作。不仅方便，而且保证了原子性。

**格式：**

- append ‘表名’，‘行键’，‘列名’，‘值’
- append ‘表名’，‘行键’，‘列名’，‘值’，ATTRIBUTES => {‘自定义键’ => ‘自定义值’}
- append ‘表名’，‘行键’，‘列名’，‘值’，{VISIBILITY => ‘PRIVATE | SECRET’}

**示例：**

~~~shell
hbase:114:0> append 'test','row4','cf:city','shanghai',ATTRIBUTES => {'test' => 'yes'}
CURRENT VALUE = shanghai
Took 0.4816 seconds                                                                               

hbase:121:0> scan 'test',{COLUMNS => ['cf:city']}
ROW                       COLUMN+CELL                                                             
 row2                     column=cf:city, timestamp=1619446123905, value=chengdu                  
 row3                     column=cf:city, timestamp=1619451161161, value=chongqing                
 row4                     column=cf:city, timestamp=1619452844704, value=shanghai                 
3 row(s)
Took 0.0106 seconds                              
~~~

#### 4.9 truncate

这个命令跟关系型数据库中同名的命令做的事情是一样的：清空表中数据，但是保留表的属性。不过 HBase truncate 表的方式其实就是先帮你删掉表，然后帮你重建表。

**格式：**

truncate ‘表名’

**示例：**

~~~shell
hbase:134:0> truncate 'table1'
Truncating 'table1' table (it may take a while):
Disabling table...
Truncating table...
Took 3.6151 seconds   
~~~

#### 4.10 truncate_preserve 

这个命令也是清空表内数据，但是它会保留表所对应的 Region。当你希望保留 Region 的拆分规则时，可以使用它，避免重新定制 Region 拆分规则。

**格式：**

truncate_preserve ‘表名’

**示例：**

~~~shell
hbase:144:0> truncate_preserve 'table1'
Truncating 'table1' table (it may take a while):
Disabling table...
Truncating table...
Took 3.0167 seconds    
~~~

#### 4.11 get_splits

获取表所对应的 Region 个数。因为一开始只有一个 Region ，由于 Region 的逐渐变大，Region 被拆分（split）为多个，所以这个命令叫 get_splits。

**格式：**

get_splits ‘表名’

**示例：**

~~~shell
hbase:146:0> get_splits 'test'
Total number of splits = 1
Took 0.0257 seconds                                                                               
=> []
~~~

### 5. 工具方法

#### 5.1 close_region

下线指定的 Region，注意，此命令对于 HBase2.x 已经淘汰，下线 Region 可以通过指定 Region 名，也可以指定 Region 名的 hash 值。那么怎么拿到 Region 名呢？你可以通过前面介绍的 locate_region 命令来获取某个行键所在的 Region。Region 名的 hash 值就是 Region 名的最后一段字符串，该字符串夹在两个句点之间。我们来一个例子。执行了 locate_region 之后的输出是这样的：

~~~shell
hbase:149:0> locate_region 'test','row1'
HOST                      REGION                                                                  
 cdp02:16020              {ENCODED => 797a96a1f98775885db2a0456f9aacb3, NAME => 'test,,16193240050
                          40.797a96a1f98775885db2a0456f9aacb3.', STARTKEY => '', ENDKEY => ''}    
1 row(s)
Took 0.0021 seconds                                                                               
=> #<Java::OrgApacheHadoopHbase::HRegionLocation:0xff5d4f1>
~~~

从输出中可以看出：

- Region 名为：

  ~~~shell
  test,,1619324005040.797a96a1f98775885db2a0456f9aacb3
  ~~~

- Region 名的 hash 值为，也即为 Region 的 ENCODED值：

  ~~~shell
  797a96a1f98775885db2a0456f9aacb3
  ~~~

- Region所在的主机以及监听的端口：

  ~~~shell
  cdp02:16020 
  ~~~

  

  **同时，你可以通过查询 hbase:meta 表知道某个 Region 的信息，就比如上面的那个 Region**

~~~shell
hbase:155:0> get 'hbase:meta','test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.'
COLUMN                    CELL                                                                    
 info:regioninfo          timestamp=1619437374527, value={ENCODED => 797a96a1f98775885db2a0456f9aa
                          cb3, NAME => 'test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.', ST
                          ARTKEY => '', ENDKEY => ''}                                             
 info:seqnumDuringOpen    timestamp=1619437374527, value=\x00\x00\x00\x00\x00\x00\x00C            
 info:server              timestamp=1619437374527, value=cdp02:16020                              
 info:serverstartcode     timestamp=1619437374527, value=1619081727904                            
 info:sn                  timestamp=1619437374162, value=cdp02,16020,1619081727904                
 info:state               timestamp=1619437374527, value=OPEN                                     
1 row(s)
Took 0.0293 seconds                 
~~~

这个 Region 对应的服务器标识码是：cdp02,16020,1619081727904 

**格式：**

- close_region ‘region名字’
- close_region ‘region名字’，‘服务器标识码’
- close_region ‘region名的hash值’，‘服务器标识码’

**示例：**

~~~shell
hbase:004:0> close_region 'test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.','1619081727904'
DEPRECATED!!! Use 'unassign' command instead.
Took 0.0007 seconds       
~~~

####5.2 unassign

下线指定的 Region。 下线指定的 Region 后马上随机找一台服务器上线该 Region。如果跟上第二个参数 true，则会强制下线，在关闭 Region 之前清空 Master 中关于该 Region 的上线状态，在某些出故障的情况下，Master 中记录的 Region 上线状态可能会跟 Region 实际的上线状态不相符，不过一般情况下你不会用到第二个参数。如果传递“ true”以强制取消分配（“ force”将清除重新分配之前，主机中的所有内存状态。 如果导致双重分配使用 hbck -fix来解决。 供专家使用）。请谨慎使用，仅供专家使用。

**格式：**

- unassign ‘region名字’
- unassign ‘region名字’，true
- unassign ‘region的ENCODED值’
- unassign ‘region的ENCODED值’，true

**示例：**

~~~shell
hbase:004:0> unassign '797a96a1f98775885db2a0456f9aacb3',true
Took 1.2022 seconds 
~~~

#### 5.3 assign

上线指定的 Region，不过如果你指定了一个已经上线的 Region 的话，这个 Region 会被强制重新上线。

**格式：**

- assign ‘region名字’
- assign ‘region名字的encode编码’

**示例：**

~~~shell
hbase:045:0> assign 'table1,,1619453458563.37e345d43d63cda533c1dc09c5df02b9.'
Took 1.4171 seconds  

hbase:057:0> assign '37e345d43d63cda533c1dc09c5df02b9'
Took 1.1193 seconds  
~~~

#### 5.4 move

移动一个 Region，你可以传入目标服务器的服务器标识码来将 Region 移动到目标服务器上。如果你不传入目标服务器标识码，那么就会将 Region 随机移动到某一个服务器上，就跟 unassign 操作的效果一样。该方法还有一个特殊的地方就是，她只接受 Region 名的 hash 值，而不是 Region名。关于 Region 名的 hash 值和服务器标识码的知识，已经在前面的 close_region命令中介绍过了，在此不再赘述。

**格式：**

- move ‘region名的hash值’，‘服务器标识码’
- move ‘region名的hash值’

**示例：**

~~~shell
# 查看test表中的row1 所在的 region名字
hbase:066:0> locate_region 'test','row1'
HOST                                     REGION                                                                                                             
 cdp02:16020                             {ENCODED => 797a96a1f98775885db2a0456f9aacb3, NAME => 'test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.', STAR
                                         TKEY => '', ENDKEY => ''}                                                                                          
1 row(s)
Took 0.0051 seconds                                                                                                                                         
=> #<Java::OrgApacheHadoopHbase::HRegionLocation:0x2d913116>

#获取 test 表中的 row1 的region 所在的机器标识符，cdp02,16020,1619081727904
hbase:068:0> get 'hbase:meta','test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.'
COLUMN                                   CELL                                                                                                               
 info:regioninfo                         timestamp=1619487263682, value={ENCODED => 797a96a1f98775885db2a0456f9aacb3, NAME => 'test,,1619324005040.797a96a1f
                                         98775885db2a0456f9aacb3.', STARTKEY => '', ENDKEY => ''}                                                           
 info:seqnumDuringOpen                   timestamp=1619487263682, value=\x00\x00\x00\x00\x00\x00\x00_                                                       
 info:server                             timestamp=1619487263682, value=cdp02:16020                                                                         
 info:serverstartcode                    timestamp=1619487263682, value=1619081727904                                                                       
 info:sn                                 timestamp=1619487259168, value=cdp02,16020,1619081727904                                                           
 info:state                              timestamp=1619487263682, value=OPEN                                                                                
1 row(s)
Took 0.0129 seconds                                                                                                                                         
 
 #移动该 region 到 cdp03,16020,1619081727753
hbase:070:0> move '797a96a1f98775885db2a0456f9aacb3','cdp03,16020,1619081727753'
Took 2.4430 seconds   

# 检查是否移动到想要的位置上
hbase:071:0> get 'hbase:meta','test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.'
COLUMN                                   CELL                                                                                                               
 info:regioninfo                         timestamp=1619489279303, value={ENCODED => 797a96a1f98775885db2a0456f9aacb3, NAME => 'test,,1619324005040.797a96a1f
                                         98775885db2a0456f9aacb3.', STARTKEY => '', ENDKEY => ''}                                                           
 info:seqnumDuringOpen                   timestamp=1619489279303, value=\x00\x00\x00\x00\x00\x00\x00f                                                       
 info:server                             timestamp=1619489279303, value=cdp03:16020                                                                         
 info:serverstartcode                    timestamp=1619489279303, value=1619081727753                                                                       
 info:sn                                 timestamp=1619489278918, value=cdp03,16020,1619081727753                                                           
 info:state                              timestamp=1619489279303, value=OPEN                                                                                
1 row(s)
Took 0.0476 seconds                                       
~~~

**注意：移动到的服务器标识符，可以直接跟上主机名和端口号就行**

~~~shell
hbase:077:0> move '797a96a1f98775885db2a0456f9aacb3','cdp02,16020'
Took 2.6768 seconds  

hbase:079:0> get 'hbase:meta','test,,1619324005040.797a96a1f98775885db2a0456f9aacb3.'
COLUMN                                   CELL                                                                                                               
 info:regioninfo                         timestamp=1619489679582, value={ENCODED => 797a96a1f98775885db2a0456f9aacb3, NAME => 'test,,1619324005040.797a96a1f
                                         98775885db2a0456f9aacb3.', STARTKEY => '', ENDKEY => ''}                                                           
 info:seqnumDuringOpen                   timestamp=1619489679582, value=\x00\x00\x00\x00\x00\x00\x00l                                                       
 info:server                             timestamp=1619489679582, value=cdp02:16020                                                                         
 info:serverstartcode                    timestamp=1619489679582, value=1619081727904                                                                       
 info:sn                                 timestamp=1619489679257, value=cdp02,16020,1619081727904                                                           
 info:state                              timestamp=1619489679582, value=OPEN                                                                                
1 row(s)
Took 0.0149 seconds  
~~~

#### 5.5 split

拆分（split）指定的 Region。除了可以等到 Region 大小达到阈值后触发自动拆分机制来拆分 Region，我们还可以手动拆分指定的 Region。通过传入切分点行键，我们可以从我们希望的切分点切分 Region。

**格式：**

- split ‘表名’
- split ‘region名’
- split ‘region名的encode值’
- split ‘表名’，‘切分点行键’
- split ‘region名’，‘切分点行键’
- split ‘region名的encode值’，‘切分点行键’

**示例：**

~~~shell
hbase:140:0> split 'table1','123_row'
Took 0.1586 seconds                                    
~~~

执行后等待几秒再查看Region信息：

~~~shell
hbase:147:0> scan 'hbase:meta',{STARTROW=>'table1',ENDROW=>'table2',COLUMNS=>['info:regioninfo']}
ROW                             COLUMN+CELL                                                                             
 table1,,1619492890981.998925ee column=info:regioninfo, timestamp=1619492892393, value={ENCODED => 998925ee5ffe164d40316
 5ffe164d4031621724b20b0b.      21724b20b0b, NAME => 'table1,,1619492890981.998925ee5ffe164d4031621724b20b0b.', STARTKEY
                                 => '', ENDKEY => '123_row'}                                                            
 table1,123_row,1619492890981.a column=info:regioninfo, timestamp=1619493045206, value={ENCODED => af738ee4f6ff984718e7f
 f738ee4f6ff984718e7fa3e4c675ea a3e4c675ea7, NAME => 'table1,123_row,1619492890981.af738ee4f6ff984718e7fa3e4c675ea7.', S
 7.                             TARTKEY => '123_row', ENDKEY => ''}                                                     
2 row(s)
Took 0.0173 seconds
~~~

可以看到 Region 被拆分为了 2个 Region了。

#### 5.6 merge_region

合并（merge）两个 Region 为一个 Region。如果传入第二个参数为 ‘true’，则会触发一次强制合并（merge）。该命令的接收参数为 Region 名或者 Region 名的 hash 值，即 Region 名最后两个句点中的那段字符串（不包含句点）。拿我们上一个命令 split 中的例子来说：

~~~shell
table1,,1619492890981.998925ee5ffe164d4031621724b20b0b.
~~~

此 Region 名的 hash 值为：

~~~shell
998925ee5ffe164d4031621724b20b0b
~~~

**格式：**

- merge_region ‘region1’，‘region2’
- merge_region ‘region1’，‘region2’, … , true
- merge_region [‘region1’,’region2’]
- merge_region ‘region1的encode值’，‘region2的encode值’,…
- merge_region ‘region1的encode值’，‘region2的encode值’,… , true
- merge_region [‘region1的encode值’，‘region2的encode值’,…]
- merge_region [‘region1的encode值’，‘region2的encode值’,…],true

**示例：**

~~~shell
hbase:159:0> merge_region '998925ee5ffe164d4031621724b20b0b', 'af738ee4f6ff984718e7fa3e4c675ea7'
Took 2.4397 seconds  
~~~

执行该命令后，然后查看该表 Region情况，可以看到现在该表只有一个 Region：

~~~shell
hbase:161:0> scan 'hbase:meta',{STARTROW=>'table1',ENDROW=>'table2',COLUMNS=>['info:regioninfo']}
ROW                                   COLUMN+CELL                                                                                                
 table1,,1619492890982.439015155a45d8 column=info:regioninfo, timestamp=1619500009630, value={ENCODED => 439015155a45d82082e39ddd8c5dd649, NAME =
 2082e39ddd8c5dd649.                  > 'table1,,1619492890982.439015155a45d82082e39ddd8c5dd649.', STARTKEY => '', ENDKEY => ''}                 
1 row(s)
Took 0.0061 seconds 
~~~

#### 5.7 compact

调用指定表的所有 Region 或者指定列族的所有 Region 的合并（compact）机制。通过 compact 机制可以合并该 Region 的列族下的所有 HFile(StoreFile)，以此来提高读取性能。**注意**，compact 跟合并（merge）并不一样。merge操作是合并 2个 Region 为 1个 Region，而 compact 操作着眼点在更小的单元，StoreFile，一个 Region 可以有一个或者多个 StoreFile，compact 操作的目的在于减少 StoreFile 的数量以增加读取性能。

**格式：**

- compact  ‘表名’
- compact  ‘region名’
- compact  ‘region名’，‘列族名’
- compact  ‘表名’，‘列族名’

**示例：**

~~~shell
hbase:027:0> compact 'table1'
Took 0.3959 seconds      
~~~

#### 5.8 major_compact

在指定的表名、region、列族上运行 major compaction。

**格式：**

- major_compact ‘表名’ 
- major_compact ‘region名’
- major_compact  ‘region名’， ‘列族名’
- major_compact ‘表名’， ‘列族名’

**示例：**

~~~shell
hbase:033:0> major_compact 'user1','cf2'
Took 0.1563 seconds
~~~

#### 5.9 compact_rs

调用指定 RegionServer 上的所有 Region 的合并机制，加上第二个参数 true，意味着执行 major compaction。

**格式：**

- compact_rs ‘服务器标识码’
- compact_rs  ‘服务器标识码’，true
- compact_rs ‘服务器主机名和端口’

**示例：**

~~~shell
hbase:043:0> compact_rs 'cdp03,16020'
Took 0.1572 seconds 
~~~

#### 5.10 balancer

手动触发平衡器（balancer）。平衡器会调整 Region 所属的服务器，让所有服务器尽量负载均衡。如果返回值为 true，说明当前集群的状况允许运行平衡器；如果返回 false，意味着有些 Region 还在执行着某些操作或者存在过渡 Region，平衡器还不能开始运行。当强制执行 balancer 的时候，可能会比修复造成更大的损失，强制均衡，仅适用于专家。

**格式：**

- balancer
- balancer "force"

**示例：**

~~~shell
hbase:045:0> balancer
true                                                                                                                                             
Took 0.1221 seconds                                                                                                                              
=> 1
~~~

#### 5.11 balance_switch

打开或者关闭平衡器。传入 true 即为打开，传入 false 即为关闭。返回值为平衡器（balancer）的前一个状态，在示例中平衡器在执行 balancer_switch 操作之前的状态是打开的。

**格式：**

- balance_switch true

- balance_switch false  

**示例：**

~~~shell
hbase:051:0> balance_switch true
Previous balancer state : true                                                                                                                   
Took 0.0421 seconds                                                                                                                              
=> "true"
~~~

#### 5.12 balancer_enabled

检测当前平衡器是否开启。

**格式：**

balancer_enabled

**示例：**

~~~shell
hbase:060:0> balancer_enabled
true                                                                                                                                             
Took 0.0294 seconds                                                                                                                              
=> true
~~~

#### 5.13 catalogjanitor_run

开始运行目录管理器（catalog janitor）。所谓的目录指的就是 hbase:meta 表中存储的 Region 信息。当 HBase 在拆分或者合并的时候，为了确保数据不丢失，都会保留原来的 Region，当拆分或者合并过程结束后再等待目录管理器来清理这些旧的 Region 信息。

**格式：**

catalogjanitor_run

**示例：**

~~~shell
hbase:001:0> catalogjanitor_run
Took 0.5792 seconds                                                                                                                              
=> 0
~~~

#### 5.14 catalogjanitor_enabled

查看当前目录管理器的开启状态。

**格式：**

catalogjanitor_enabled

**示例：**

~~~shell
hbase:064:0> catalogjanitor_enabled
true                                                                                                                                             
Took 0.0228 seconds                                                                                                                              
=> 1
~~~

#### 5.15 catalogjanitor_switch

启用或者停用目录管理器。该命令会返回命令执行后状态的前一个状态。

**格式：**

- catalogjanitor_switch true
- catalogjanitor_switch false

**示例：**

~~~
hbase:002:0> catalogjanitor_switch true
true                                                                                                                                             
Took 0.0374 seconds                                                                                                                              
=> 1
~~~

#### 5.16 normalize

规整器用于规整 Region 的尺寸，通过该命令可以手动启动规整器。只有 NORMALIZE_ENABLED 为 true 的表才会参与规整过程。如果返回 true，则说明规整器启动成功；如果某个 Region 被设置为禁用规整器，则该命令不会对其产生任何效果。

**格式：**

normalize

**示例：**

~~~shell
hbase:001:0> normalize
true                                                                                                                                             
Took 0.6325 seconds                                                                                                                              
=> 1
~~~

#### 5.17 normalizer_enabled

查看规整器的启用，停用状态

**格式：**

normalizer_enabled

**示例：**

~~~shell
hbase:007:0> normalizer_enabled
true                                                                                                                                             
Took 0.0346 seconds                                                                                                                              
=> 1
~~~

#### 5.18 normalizer_switch

启用，停用规整器，该命令会返回规整器的前一个状态。

**格式：**

- normalizer_switch true
- normalizer_switch false 

**示例：**

~~~shell
hbase:009:0> normalizer_switch true
true                                                                                                                                             
Took 0.0741 seconds                                                                                                                              
=> 1
~~~

#### 5.19 flush

手动触发指定表或者 Region 的刷写。所谓的刷写就是将 memstore 内的数据持久化到磁盘上，称为 HFile 文件。

**格式：**

- flush ‘表名’
- flush ‘region名’
- flush ‘region的encode值’
- flush ‘region_server_name’

**示例：**

~~~shell
hbase:015:0> flush 'table1'
Took 0.8372 seconds  
~~~

#### 5.20 trace

启用或者关闭 trace 功能。不带任何参数的执行该命令会返回 trace 功能的开启或者关闭状态。当第一个参数使用 ‘start’ 的候，会创建新的 trace 段（trace span）：如果传入第二个参数还可以指定 trace 段的名称，否则默认使用 ‘HBaseShell’ 作为 trace 段的名称；当第一个参数传入 ‘stop’ 的时候，当前 trace 段会被关闭；当第一个参数使用 ‘status’ 的时候，会返回当前 trace 功能的开启或者关闭状态。

**格式：**

- trace 
- trace 'start'
- trace 'status'
- trace 'stop'
- trace 'start', 'trace段名称'

**示例：**

~~~shell
hbase:001:0> trace
Took 0.0019 seconds                                                                                                                              
=> false
hbase:002:0> trace 'status'
Took 0.0016 seconds                                                                                                                              
=> false
hbase:003:0> trace 'start'
Took 0.3022 seconds                                                                                                                              
=> true
hbase:004:0> create 'table2','cf2'
Created table table2
Took 2.9608 seconds                                                                                                                              
=> Hbase::Table - table2
hbase:005:0> trace 'stop'
Took 0.0005 seconds                                                                                                                              
=> false
~~~

#### 5.21 wal_roll

手动触发 WAL 的滚动

**格式：**

wal_roll ‘服务器标识码’ 。注意：服务器标识码，可以直接在HBase Web UI 页面查看。

![image-20210427202002850](img/HBase/hbase shell.png)

**示例：**

~~~shell
hbase:012:0> wal_roll 'cdp03,16020,1619081727753'
Took 0.0851 seconds  
~~~

#### 5.22 zk_dump

打印出 Zookeeper 集群中存储的 HBase 集群信息

**格式：**

zk_dump

**示例：**

~~~shell
hbase:013:0> help 'zk_dump'
Dump status of HBase cluster as seen by ZooKeeper.
hbase:014:0> zk_dump
HBase is rooted at /hbase
Active master address: cdp01,16000,1619513268195
Backup master addresses:
Region server holding hbase:meta: cdp02,16020,1619081727904
Region servers:
 cdp02,16020,1619081727904
 cdp01,16020,1619081724381
 cdp03,16020,1619081727753
Quorum Server Statistics:
 cdp02:2181
  Zookeeper version: 3.5.5-257-85741307d7893005ce1ee54292142471e6f2e9c0, built on 11/26/2020 15:30 GMT
  Clients:
   /192.168.0.128:35282[1](queued=0,recved=22522,sent=22560)
   /192.168.0.131:34126[1](queued=0,recved=26875,sent=26875)
   /192.168.0.128:53524[1](queued=0,recved=1546,sent=1567)
   /192.168.0.128:53630[1](queued=0,recved=715,sent=715)
   /192.168.0.128:59548[0](queued=0,recved=1,sent=0)
   /192.168.0.133:37366[1](queued=0,recved=29860,sent=29860)
   /192.168.0.131:56324[1](queued=0,recved=29962,sent=29962)
   /192.168.0.128:51172[1](queued=0,recved=33850,sent=33919)
   /192.168.0.128:35402[1](queued=0,recved=29939,sent=29939)
   /192.168.0.128:54664[1](queued=0,recved=9887,sent=9887)
   /192.168.0.128:51252[1](queued=0,recved=19082,sent=19082)
  
  Latency min/avg/max: 0/0/744
  Received: 4042921
  Sent: 4308011
  Connections: 11
  Outstanding: 0
  Zxid: 0xd0013d2af
  Mode: follower
  Node count: 1476
 cdp01:2181
  Zookeeper version: 3.5.5-257-85741307d7893005ce1ee54292142471e6f2e9c0, built on 11/26/2020 15:30 GMT
  Clients:
   /192.168.0.128:44468[1](queued=0,recved=688,sent=688)
   /192.168.0.131:53498[1](queued=0,recved=22309,sent=22309)
   /192.168.0.133:49114[1](queued=0,recved=22466,sent=22499)
   /192.168.0.128:55898[1](queued=0,recved=1093,sent=1093)
   /192.168.0.131:38584[1](queued=0,recved=6741,sent=6741)
   /192.168.0.128:49312[1](queued=0,recved=27086,sent=27086)
   /192.168.0.131:57566[1](queued=0,recved=22480,sent=22512)
   /192.168.0.128:59974[1](queued=0,recved=22308,sent=22308)
   /192.168.0.128:33488[1](queued=0,recved=8,sent=8)
   /192.168.0.128:59948[1](queued=0,recved=23201,sent=23201)
   /192.168.0.133:51150[1](queued=0,recved=29886,sent=29886)
   /192.168.0.131:53502[1](queued=0,recved=22304,sent=22304)
   /192.168.0.133:56388[1](queued=0,recved=29865,sent=29867)
   /192.168.0.128:53428[1](queued=0,recved=19082,sent=19082)
   /192.168.0.131:37756[1](queued=0,recved=10197,sent=10197)
   /192.168.0.131:44612[1](queued=0,recved=33614,sent=33615)
   /192.168.0.128:56750[1](queued=0,recved=10514,sent=10519)
   /192.168.0.128:33492[0](queued=0,recved=1,sent=0)
  
  Latency min/avg/max: 0/0/1211
  Received: 3015744
  Sent: 3281052
  Connections: 18
  Outstanding: 0
  Zxid: 0xd0013d2af
  Mode: leader
  Node count: 1476
  Proposal sizes last/min/max: 36/32/55976
 cdp03:2181
  Zookeeper version: 3.5.5-257-85741307d7893005ce1ee54292142471e6f2e9c0, built on 11/26/2020 15:30 GMT
  Clients:
   /192.168.0.128:59222[1](queued=0,recved=22299,sent=22299)
   /192.168.0.128:60972[0](queued=0,recved=1,sent=0)
   /192.168.0.131:41330[1](queued=0,recved=19082,sent=19082)
   /192.168.0.131:41322[1](queued=0,recved=19081,sent=19081)
   /192.168.0.131:39294[1](queued=0,recved=9958,sent=9958)
   /192.168.0.133:52308[1](queued=0,recved=10195,sent=10195)
   /192.168.0.133:52346[1](queued=0,recved=9998,sent=9998)
   /192.168.0.128:50420[1](queued=0,recved=33616,sent=33617)
   /192.168.0.128:49922[1](queued=0,recved=26745,sent=26745)
   /192.168.0.131:48292[1](queued=0,recved=81520,sent=81521)
   /192.168.0.131:48310[1](queued=0,recved=33774,sent=33833)
   /192.168.0.128:52590[1](queued=0,recved=386606,sent=386619)
   /192.168.0.131:54900[1](queued=0,recved=22302,sent=22302)
  
  Latency min/avg/max: 0/1/1801
  Received: 3067885
  Sent: 3333050
  Connections: 13
  Outstanding: 0
  Zxid: 0xd0013d2af
  Mode: follower
  Node count: 1476
Took 0.6912 seconds                                                
~~~

### 6. 快照

#### 6.1 snapshot

快照（snapshot）就是表在某个时刻的结构和数据。可以使用快照来将某个表恢复到某个时刻的结构和数据。通过 snapshot 命令可以创建指定表的快照。

**格式：**

- snapshot ‘表名’， ‘快照名’
- snapshot ‘表名’， ‘快照名’， {SKIP_FLUSH => true}

**示例：**

~~~shell
hbase:017:0> snapshot 'table1','table1_snapshot', {SKIP_FLUSH => true}
Took 1.4874 seconds                                                                                                                                          
=> [{"SKIP_FLUSH"=>true}]
hbase:018:0> list_snapshots
SNAPSHOT                                 TABLE + CREATION TIME                                                                                               
 table1_snapshot                         table1 (2021-04-27 20:40:11 +0800)                                                                                  
1 row(s)
Took 0.0541 seconds                                                                                                                                          
=> ["table1_snapshot"]
~~~

#### 6.2 list_snapshots

列出所有快照，可以传入正则表达式来查询快照列表。

**格式：**

- list_snapshots
- list_snapshots ‘正则表达式’

**示例：**

~~~shell
hbase:024:0> list_snapshots 'table.*'
SNAPSHOT                                 TABLE + CREATION TIME                                                                                               
 table1_snapshot                         table1 (2021-04-27 20:40:11 +0800)                                                                                  
 table2_snapshot                         table2 (2021-04-27 20:42:56 +0800)                                                                                  
2 row(s)
Took 0.0282 seconds                                                                                                                                          
=> ["table1_snapshot", "table2_snapshot"]
~~~

#### 6.3 restore_snapshot

使用快照恢复表。由于表的数据会被全部重复，所在在根据快照恢复表之前，必须要停用该表。

**格式：**

- restore_snapshot ‘快照名’
- restore_snapshot ‘快照名’ , {RESTORE_ACL=>true}

**示例：**

~~~shell
hbase:026:0> disable 'table1'
Took 0.8421 seconds                                                                                                                                          
hbase:027:0> restore_snapshot 'table1_snapshot'
Took 1.1359 seconds                                                                                                                                          
hbase:028:0> enable 'table1'
Took 1.3020 seconds         
~~~

#### 6.4 clone_snapshot

使用快照的数据创建出一张新表。创建的过程很快，因为使用的方式不是复制数据，并且修改新表的数据也不会影响旧表的数据。

**格式：**

- clone_snapshot ‘快照名’， ‘新表名’
- clone_snapshot  '快照名', '新表名',  {RESTORE_ACL=>true}

**示例：**

~~~shell
hbase:034:0> clone_snapshot 'table1_snapshot', 'tableA'
Took 2.3205 seconds   

hbase:035:0> count 'tableA'
1 row(s)
Took 0.0109 seconds                                                                                                                                          
=> 1
~~~

#### 6.5 delete_snapshot

删除快照

**格式：**

delete_snapshot ‘快照名’

**示例：**

~~~shell
hbase:040:0> delete_snapshot 'table1_snapshot'
Took 0.0239 seconds       
~~~

#### 6.6 delete_all_snapshot

同时删除多个跟正则表达式匹配的快照

**格式：**

delete_all_snapshot  ‘正则表达式’

**示例：**

~~~shell
hbase:056:0> delete_all_snapshot 'table.*'
SNAPSHOT                                 TABLE + CREATION TIME                                                                                               
 table1_snapshot                         table1 (2021-04-27 20:58:56 +0800)                                                                                  
 table2_snapshot                         table2 (2021-04-27 20:59:05 +0800)                                                                                  

Delete the above 2 snapshots (y/n)?
y
2 snapshots successfully deleted.
Took 0.0398 seconds                    
~~~

### 7. 命名空间

#### 7.1 list_namespace

列出所有的命令空间，你还可以通过传入正则表达式来过滤结果。

**格式：**

- list_namespace
- list_namespace  ‘正则表达式’

**示例***

~~~shell
hbase:061:0> list_namespace
NAMESPACE                                                                                                                                                    
default                                                                                                                                                      
hbase                                                                                                                                                        
2 row(s)
Took 0.0669 seconds   
~~~

#### 7.2 list_namespace_tables

列出该命名空间下的表

**格式：**

list_namespace_tables  ‘命令空间’

**示例：**

~~~shell
hbase:066:0> list_namespace_tables 'hbase'
TABLE                                                                                                                                                        
acl                                                                                                                                                          
meta                                                                                                                                                         
namespace                                                                                                                                                    
3 row(s)
Took 0.0064 seconds                                                                                                                                          
=> ["acl", "meta", "namespace"]
~~~

#### 7.4 create_namespace

创建命令空间，你还可以在创建命令空间的同时指定属性。

**格式：**

- create_namespace  ‘命名空间名’
- create_namespace  ‘命名空间名’ ， {‘属性名’  => ‘属性值’}

**示例：**

~~~shell
hbase:069:0> create_namespace 'testnamespace'
Took 0.3247 seconds     

hbase:070:0> list_namespace
NAMESPACE                                                                                                                                                    
default                                                                                                                                                      
hbase                                                                                                                                                        
testnamespace                                                                                                                                                
3 row(s)
Took 0.0250 seconds  
~~~

#### 7.5 describe_namespace

显示命名空间定义

**格式：**

describe_namespace ‘命令空间’

**示例：**

~~~shell
hbase:075:0> describe_namespace 'default'
DESCRIPTION                                                                                                                                                  
{NAME => 'default'}                                                                                                                                          
Quota is disabled
Took 0.0075 seconds   
~~~

#### 7.6 alter_namespace

更改命名空间的属性或者删除该属性。如果 METHOD 使用 set 表示设定属性，使用 unset 表示删除属性。

**格式：**

- alter_namespace  ‘命名空间名’， {METHOD => ‘set’, ‘属性名’  => ‘属性值’}
- alter_namespace  ‘命名空间名’， {METHOD => ‘unset’, NAME  => ‘属性值’}

**示例：**

~~~shell
hbase:079:0> alter_namespace 'testnamespace', {METHOD => 'set', 'att1' => 'value1'}
Took 0.3736 seconds     

hbase:080:0> describe_namespace 'testnamespace'
DESCRIPTION                                                                                                                                                  
{NAME => 'testnamespace', att1 => 'value1'}                                                                                                                  
Quota is disabled
Took 0.0233 seconds                                                                                                                                          

hbase:084:0> alter_namespace 'testnamespace', {METHOD => 'unset', NAME => 'value1'}
Took 0.2377 seconds                                                                                                                                          

hbase:087:0> describe_namespace 'testnamespace'
DESCRIPTION                                                                                                                                                  
{NAME => 'testnamespace', att1 => 'value1'}                                                                                                                  
Quota is disabled
Took 0.0082 seconds                                      
~~~

#### 7.7 drop_namespace

删除命名空间。不过在删除之前，请先确保命名空间内没有表，否则你会得到以下报错信息：

~~~shell
hbase:100:0> drop_namespace 'testnamespace'

ERROR: org.apache.hadoop.hbase.constraint.ConstraintException: Only empty namespaces can be removed. Namespace testnamespace has 1 tables
        at org.apache.hadoop.hbase.master.procedure.DeleteNamespaceProcedure.prepareDelete(DeleteNamespaceProcedure.java:217)
        at org.apache.hadoop.hbase.master.procedure.DeleteNamespaceProcedure.executeFromState(DeleteNamespaceProcedure.java:78)
        at org.apache.hadoop.hbase.master.procedure.DeleteNamespaceProcedure.executeFromState(DeleteNamespaceProcedure.java:45)
        at org.apache.hadoop.hbase.procedure2.StateMachineProcedure.execute(StateMachineProcedure.java:194)
        at org.apache.hadoop.hbase.procedure2.Procedure.doExecute(Procedure.java:962)
        at org.apache.hadoop.hbase.procedure2.ProcedureExecutor.execProcedure(ProcedureExecutor.java:1662)
        at org.apache.hadoop.hbase.procedure2.ProcedureExecutor.executeProcedure(ProcedureExecutor.java:1409)
        at org.apache.hadoop.hbase.procedure2.ProcedureExecutor.access$1100(ProcedureExecutor.java:78)
        at org.apache.hadoop.hbase.procedure2.ProcedureExecutor$WorkerThread.run(ProcedureExecutor.java:1979)

For usage try 'help "drop_namespace"'

Took 0.1480 seconds                                                                                                                                          
hbase:101:0> list_namespace_tables 'testnamespace'
TABLE                                                                                                                                                        
table                                                                                                                                                        
1 row(s)
Took 0.0259 seconds                                                                                                                                          
=> ["table"]
~~~

**格式：**

drop_namespace ‘命名空间名’

**示例：**

~~~shell
hbase:088:0> drop_namespace 'testnamespace1'
Took 0.3025 seconds   
~~~

### 8. 配置

#### 8.1 update_config

要求制定服务器重新加载配置文件。参数为服务器标识码。

**格式：**

update_config ‘服务器标识码’

**示例：**

~~~shell
hbase:108:0> update_config 'cdp01,16020,1619081724381'
Took 0.2529 seconds     
~~~

#### 8.2 update_all_config

要求所有服务器重新加载配置文件

**格式：**

update_all_config

**示例：**

~~~shell
hbase:110:0> update_all_config
Took 1.8455 seconds    
~~~

### 9. 集群备份

#### 9.1 add_peer

一个备份节点（peer节点）可以是一个 HBase 集群，也可以是你自定义的一套存储方案。

**格式：**

- add_peer ‘peer id’ ,CLUSTER_KEY => "zk的ip和端口/hbase在zk里面的根节点"
- add_peer ‘peer id’ ,CLUSTER_KEY => "zk的ip和端口/hbase在zk里面的根节点"，TABLE_CFS => {要备份的表或者列族}
- add_peer ‘peer id’ ,CLUSTER_KEY => "zk的ip和端口/hbase在zk里面的根节点"，“要备份的表或者列族”

**示例：**

~~~shell
hbase:134:0> add_peer '1', CLUSTER_KEY => "cdp01:2181:/hbase"
Took 2.3237 seconds            
~~~

#### 9.2 remove_peer_tableCFs

从指定节点的配置中删除指定表或者列族信息，这样该表或者列族将不再参与备份操作

**格式：**

remove_peer_tableCFs ‘peer id’ ,”表或者列族名”

**示例：**

~~~shell
hbase:153:0> remove_peer_tableCFs '1',  {"table1" => ["cf"]
~~~

#### 9.3 set_peer_tableCFs

设定指定备份节点的备份表或者列族信息

**格式：**

set_peer_tableCFs '2',{ "ns1:table1" => [], "ns2:table2" => ["cf1", "cf2"],"ns3:table3" => ["cfA", "cfB"]}

**示例：**

~~~shell
hbase:005:0> set_peer_tableCFs '1',{ "table2" => []}
~~~

#### 9.4 show_peer_tableCFs

列出指定备份节点的备份表或者列族信息

**格式：**

show_peer_tableCFs ‘peer id’

**示例：**

~~~shell
hbase:009:0> show_peer_tableCFs '1'
/opt/cloudera/parcels/CDH-7.1.5-1.cdh7.1.5.p0.7431829/lib/hbase/lib/ruby/hbase/replication_admin.rb:162: warning: multiple Java methods found, use -Xjruby.ji.ambiguous.calls.debug for backtrace. Choosing convertToString(java.util.Map)

Took 0.0065 seconds   
~~~

#### 9.5 append_peer_tableCFs

为现有的备份节点（peer节点）配置增加新的列族

**格式：**

append_peer_tableCFs '2', { "ns1:table4" => ["cfA", "cfB"]}

**示例：**

~~~shell
hbase:024:0> append_peer_tableCFs '1', { "table1" => ["cfA", "cfB"]}
~~~

#### 9.6 disable_peer

终止向指向集群发送备份数据

**格式：**

disable_peer ‘peer id’

**示例：**

~~~shell
hbase:025:0> disable_peer '1'
~~~

#### 9.7 enable_peer

disable_peer 操作之后，重新启用知道备份节点的备份操作

**格式：**

enable_peer ‘peer id’

**示例：**

~~~shell
hbase:004:0> enable_peer '1'
~~~

#### 9.8 disable_table_replication

取消指定表的备份操作

**格式：**

disable_table_replication ‘表名’

**示例：**

~~~shell
hbase:002:0> disable_table_replication 'table1'
Replication of table 'table1' successfully disabled.
Took 0.3668 seconds                
~~~

#### 9.9 enable_table_replication

disable_table_replication  操作之后，重新启用指定表的备份操作

**格式：**

enable_table_replication ‘表名’

**示例：**

~~~shell
hbase:003:0> enable_table_replication 'table1'
The replication of table 'table1' successfully enabled
Took 4.0524 seconds     
~~~

#### 9.10 list_peers

列出所有备份节点

**格式：**

list_peers

**示例：**

~~~shell
hbase:004:0> list_peers
 PEER_ID CLUSTER_KEY ENDPOINT_CLASSNAME STATE REPLICATE_ALL NAMESPACES TABLE_CFS BANDWIDTH SERIAL
 1 cdp01:2181:/hbase  DISABLED true   0 false
1 row(s)
Took 0.0102 seconds                                                                                                                                          
=> #<Java::JavaUtil::ArrayList:0x4b691611>
~~~

#### 9.11 list_replicated_tables

列出所有参加备份操作的表或者列族

**格式：**

- list_replicated_tables
- list_replicated_tables ‘正则表达式’

**示例：**

~~~shell
hbase:010:0> list_replicated_tables
TABLE:COLUMNFAMILY                                  ReplicationType                                                                                          
 table1:cf1                                         GLOBAL                                                                                                   
1 row(s)
Took 0.0253 seconds   
~~~

#### 9.12 remove_peer

停止指定备份节点，并删除所有该节点关联的备份元数据

**格式：**

remove_peer ‘peer id’

**示例：**

~~~shell
hbase:020:0> remove_peer '1'
~~~

### 10. 安全

#### 10.1 list_security_capabilities

列出所有支持的安全特性

**格式：**

list_security_capabilities

**示例：**

~~~shell
hbase:001:0> list_security_capabilities
AUTHORIZATION
SECURE_AUTHENTICATION
CELL_AUTHORIZATION
Took 0.6364 seconds                                                                                                                                          
=> ["AUTHORIZATION", "SECURE_AUTHENTICATION", "CELL_AUTHORIZATION"]
~~~

#### 10.2 user_permission

列出指定用户权限，或者指定用户针对指定表的权限。如果要表示整个命名空间，而不特指某张表，请用 @命名空间

**格式：**

- user_permission
- user_permission ‘表名’

**示例：**

~~~shell
hbase:002:0> user_permission
User                                     Namespace,Table,Family,Qualifier:Permission                                                                         
 hbase                                   ,,,: [Permission: actions=READ,WRITE,EXEC,CREATE,ADMIN]                                                             
1 row(s)
Took 0.5794 seconds                                                                                                                                          
hbase:003:0> user_permission 'table1'
User                                     Namespace,Table,Family,Qualifier:Permission                                                                         
0 row(s)
Took 0.1116 seconds                                                                                                                                          

hbase:005:0> user_permission '@hbase'
User                                     Namespace,Table,Family,Qualifier:Permission                                                                         
0 row(s)
Took 0.2444 seconds                                                           
~~~

#### 10.3 grant

赋予用户权限。可选的权限有：

- READ(‘R’)
- WRITE(‘W’)
- EXEC(‘X’)
- CREATE(‘C’)
- ADMIN(‘A’)

同样的，如果要表示整个命名空间，而不特指某张表，请用@命名空间名

**格式：**

- grant ‘用户’， ‘权限表达式’
- grant ‘用户’， ‘权限表达式’，‘表名’
- grant ‘用户’， ‘权限表达式’，‘表名’，‘列族名’
- grant ‘用户’， ‘权限表达式’，‘表名’，‘列族名’，‘列名’

**示例：**

~~~
hbase:018:0> grant '@hbase', 'RWXCA'
Took 1.2181 seconds              

hbase:012:0> grant 'testuser', 'RWXCA'
Took 1.3512 seconds    

hbase:022:0> grant 'testuser', 'RWXCA','table1','cf1','city'
Took 0.7898 seconds       
~~~

#### 10.4 revoke

取消用户的权限。如果要表示整个命名空间，而不特指某张表，请用@命名空间

**格式：**

- revoke ‘用户’，‘权限表达式’
- revoke ‘用户’，‘权限表达式’，‘表名’
- revoke ‘用户’，‘权限表达式’，‘表名’，‘列族名’
- revoke ‘用户’，‘权限表达式’，‘表名’，‘列族名’，‘列名’

**示例：**

~~~shell
hbase:027:0> revoke 'testuser'
Took 1.1454 seconds                                                                                                                                          

hbase:029:0> revoke '@hbase'
Took 0.8896 seconds                                                                                                                                          

hbase:032:0> revoke 'testuser','table1','cf1','city'
Took 0.4513 seconds                            
~~~

### 11. 总结

通过上面的这些常用的hbase shell 命令操作，我们可以更深入的了解 hbase 的特性，以及 hbase 的原理，要想研究透一个东西，第一步总是先用起来，从使用的过程逐渐深入的掌握其背后的原理。
