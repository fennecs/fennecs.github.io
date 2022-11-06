---
title: '[Mysql]漫游bin log'
categories:
  - Mysql
slug: 2414692924
date: 2020-05-26 09:51:00
tags:
---
binlog是server层的日志，对于innodb来说，只有binlog写完后，才能提交redo log。binlog记录逻辑语句，只会记录写类似于sql的日志。binlog主要的作用
1. 崩溃恢复
2. 主从复制

# 组织结构
## 文件
binlog的相关配置可以执行`show variables like '%bin%';`查看
```sql
mysql> show variables like '%bin%';
+--------------------------------------------+---------------------------------+
| Variable_name                              | Value                           |
+--------------------------------------------+---------------------------------+
| bind_address                               | *                               |
| binlog_cache_size                          | 32768                           |
| binlog_checksum                            | CRC32                           |
| binlog_direct_non_transactional_updates    | OFF                             |
| binlog_error_action                        | ABORT_SERVER                    |
| binlog_format                              | ROW                             |
| binlog_group_commit_sync_delay             | 0                               |
| binlog_group_commit_sync_no_delay_count    | 0                               |
| binlog_gtid_simple_recovery                | ON                              |
| binlog_max_flush_queue_time                | 0                               |
| binlog_order_commits                       | ON                              |
| binlog_row_image                           | FULL                            |
| binlog_rows_query_log_events               | OFF                             |
| binlog_stmt_cache_size                     | 32768                           |
| binlog_transaction_dependency_history_size | 25000                           |
| binlog_transaction_dependency_tracking     | COMMIT_ORDER                    |
| innodb_api_enable_binlog                   | OFF                             |
| innodb_locks_unsafe_for_binlog             | OFF                             |
| log_bin                                    | ON                              |
| log_bin_basename                           | /data/mysql3306/mysql-bin       |
| log_bin_index                              | /data/mysql3306/mysql-bin.index |
| log_bin_trust_function_creators            | ON                              |
| log_bin_use_v1_row_events                  | OFF                             |
| log_statements_unsafe_for_binlog           | ON                              |
| max_binlog_cache_size                      | 18446744073709547520            |
| max_binlog_size                            | 536870912                       |
| max_binlog_stmt_cache_size                 | 18446744073709547520            |
| sql_log_bin                                | ON                              |
| sync_binlog                                | 1                               |
+--------------------------------------------+---------------------------------+
```

执行`show master status;`可以看到当前写入的二进制文件的名字和位置。
```sql
mysql> show master status;
+------------------+-----------+--------------+------------------+-------------------+
| File             | Position  | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+-----------+--------------+------------------+-------------------+
| mysql-bin.000930 | 178613715 |              | mysql,test       |                   |
+------------------+-----------+--------------+------------------+-------------------+
1 row in set (0.10 sec)
```


binlog主要有两种文件：
* 二进制日志索引文件（文件名后缀为.index）用于记录所有有效的的二进制文件。
* 二进制日志文件（文件名后缀为.****）记录数据库所有的DDL和DML语句事件

如上输出，二进制日志文件存放的`basename`是"/data/mysql3306/mysql-bin.****"，二进制日志索引文件路径是"/data/mysql3306/mysql-bin.index",

具体如下
```shell
[root@dev_test63 mysql3306]# ls -l mysql-bin*
-rw-r----- 1 mysql mysql 538582976 5月  22 01:17 mysql-bin.000919
-rw-r----- 1 mysql mysql 536879010 5月  22 12:03 mysql-bin.000920
-rw-r----- 1 mysql mysql 536885288 5月  22 20:03 mysql-bin.000921
-rw-r----- 1 mysql mysql 536871917 5月  23 05:05 mysql-bin.000922
-rw-r----- 1 mysql mysql 536871391 5月  23 21:40 mysql-bin.000923
-rw-r----- 1 mysql mysql 536895385 5月  24 07:03 mysql-bin.000924
-rw-r----- 1 mysql mysql 536945072 5月  25 00:03 mysql-bin.000925
-rw-r----- 1 mysql mysql 536904757 5月  25 09:03 mysql-bin.000926
-rw-r----- 1 mysql mysql 536871742 5月  25 19:28 mysql-bin.000927
-rw-r----- 1 mysql mysql 536870960 5月  26 04:34 mysql-bin.000928
-rw-r----- 1 mysql mysql 536871255 5月  26 19:33 mysql-bin.000929
-rw-r----- 1 mysql mysql 153202413 5月  27 00:22 mysql-bin.000930
-rw-r----- 1 mysql mysql       228 5月  26 19:33 mysql-bin.index
```
`sync_binlog`是用来表示同步刷binlog，如果数据库繁忙可能会造成磁盘io压力大

1. 参数为0时，并不是立即fsync文件到磁盘，而是依赖于操作系统的fsync机制；
2. 参数为1时，立即fsync文件到磁盘；
3. 参数大于1时，则达到指定提交次数后，统一fsync到磁盘。 因此只有当sync_binlog参数为1时，才是最安全的，当其不为1时，都存在binlog未fsync到磁盘的风险，若此时发生断电等故障，就有可能出现此事务并未刷出到磁盘。

`sql_log_bin`表示开启binlog，开关都需要重启mysql
**mysql-bin.index**这个文件很简单，只是记录了当前的binlog列表
```shell
[root@dev_test63 mysql3306]# cat mysql-bin.index
./mysql-bin.000919
./mysql-bin.000920
./mysql-bin.000921
./mysql-bin.000922
./mysql-bin.000923
./mysql-bin.000924
./mysql-bin.000925
./mysql-bin.000926
./mysql-bin.000927
./mysql-bin.000928
./mysql-bin.000929
./mysql-bin.000930
```
而binlog是一个二进制文件集合，每个binlog文件以一个4字节的魔数`0xfe62696e`开头（后3字节其实就是"bin"）

接着是一组Events，Event由header组成，header记录了创建时间、服务器标识等，data是数据。一个binlog文件里，第一个event描述binlog的格式，最后一个binlog描述下一个binlog文件的信息。

## rotation
当下面三种情况发生时，binlog会rotate新文件：
* 实例停止或重启时
* flush logs 命令；
* 当前binlog > `max_binlog_size`(像上面的配置是512M)

> 如果有一个大事务执行时，很可能会发生一个binlog文件稍大于`max_binlog_size`

# 模式
binlog有三种记录模式，分别是**statement**，**row**，**mixed**，通过`binlog_format`来指定，可以在运行时指定。
<!--TODO mysql binlog设置-->

## statement
**statement**记录的是原语句，你执行什么语句就会记录什么语句。这在主从复制的时候就会出现一些问题：

这里提一个问题：**为什么大多数数据库的默认隔离级别是RC，而innodb是RR呢**：

这是一个历史遗留问题，网上也可以找到很多解释，大体就是：在mysql5.1.5之前只有**statement**模式，如果事务用RC隔离级别，就会可能出现主从不一致的情况。

<!--TODO 验证-->
现在已经不能在**statement**模式下执行RC事务了，会报下面的错误：
> Cannot execute statement: impossible to write to binary log since BINLOG_FORMAT = STATEMENT and at least one table uses a storage engine limited to row-based logging. InnoDB is limited to row-logging when transaction isolation level is READ COMMITTED or READ UNCOMMITTED.

假设RC允许，那么假设student表有score字段，

| 事务A                                | 事务B                                  |
|------------------------------------|-----------------------------------------|
| begin;                             | begin;                                  |
| delete from student where score < 6; | /                                       |
| /                                  | insert into student(score) values(1);     |
| /                                  | commit;                                 |
| commit;                            | /                                       |

在RR级别下，事务B的语句会阻塞，因为事务A会给满足条件的列加上X锁，给间隙加上gap锁，所以事务B是会被阻塞的。
在RC级别下，事务B的语句是不会阻塞的，因此先于事务A提交，提交完成写入binlog，接着A提交写入binlog，

所以在binlog里是这样的（只是举个🌰）：
```sql
insert into student(age) values(1);
delete from student where age < 6;
```
这样binlog被从库消费之后就主从就不一致。

## row
**row**是记录到每一行的逻辑语句，比如执行`update student set name = '土川' where age < 10`的sql，就会生成n条记录。

这样可以避免**statement**产生的问题，**缺点**就是日志量太大，可能会产生io压力。
## mixed
**mixed**模式是由mysql自行判断使用哪种模式。转换条件[传送门](https://dev.mysql.com/doc/refman/5.7/en/binary-log-mixed.html)

> 新版本的MySQL对row level模式也被做了优化，并不是所有的修改都会以row level来记录，像遇到表结构变更的时候就会以statement模式来记录，如果sql语句确实就是update或者delete等修改数据的语句，那么还是会记录所有行的变更；因此，现在一般使用row level即可。

# binlog内容
## mysqlbinlog
mysqlbinlog是mysql提供的一个查看binlog的工具，通过该工具可以将二进制文件解析为文本供我们进行阅读，还可以查看远程服务器上的binlog，可以指定时间或偏移量作为start、stop来作为查询条件。如果指定的偏移量不是一个event的起始偏移量，则会报错。

当前msyql是row模式，执行` mysqlbinlog -v --start-datetime="2020-05-25 09:59:59" --stop-datetime="2020-05-25 10:00:00" mysql-bin.000927 --base64-output=decode-rows`，`--base64-output=decode-rows`为了解码`row`格式，`-v`详细输出语句。截取一部分如下
```sql
# at 26237733
#200525  9:59:59 server id 3306  end_log_pos 26237798 CRC32 0xbcd552f0 	GTID [commit=no]
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 26237798
#200525  9:59:59 server id 3306  end_log_pos 26237883 CRC32 0x5558cbbe 	Query	thread_id=43976966	exec_time=0	error_code=0
SET TIMESTAMP=1590371999/*!*/;
BEGIN
/*!*/;
# at 26237883
#200525  9:59:59 server id 3306  end_log_pos 26237968 CRC32 0x8f9f400f 	Table_map: `superq_db`.`superq_trigger_registry` mapped to number 96550
# at 26237968
#200525  9:59:59 server id 3306  end_log_pos 26238162 CRC32 0x560afdaa 	Update_rows: table id 96550 flags: STMT_END_F
### UPDATE `superq_db`.`superq_trigger_registry`
### WHERE
###   @1=15093
###   @2='EXECUTOR'
###   @3='job-executor-saber'
###   @4='172.88.2.128:19012'
###   @5='172.88.2.128:10099'
###   @6=1590371989
### SET
###   @1=15093
###   @2='EXECUTOR'
###   @3='job-executor-saber'
###   @4='172.88.2.128:19012'
###   @5='172.88.2.128:10099'
###   @6=1590371999
# at 26238162
#200525  9:59:59 server id 3306  end_log_pos 26238193 CRC32 0x1e662746 	Xid = 3290666026
COMMIT/*!*/;
```
* `# at xxxx`：一个event的开头，
* `#200525  9:59:59`：时间，20年5月25日，9时59分59秒
* `server id 3306`：server编号
* `end_log_pos 26237798`：下一个事件开始的位置（即当前事件的结束位置+1）
* `CRC32 0xbcd552f0`：CRC32是校验和的算法，在上面配置清单里`binlog_checksum`可以看到，后面跟着的是32位校验和。
* `Query`：event type，具体可以翻阅[Event Classes and Types](https://dev.mysql.com/doc/internals/en/event-classes-and-types.html)
* `thread_id=43976966`：线程id
* `exec_time=0`：执行时间
* `error_code=0`：错误码，0表示无错误
* `Xid = 3290666026`：表示redo log和binlog做XA的Xid

### 结构体
前面说到，event分为header和data，事实上，binlog 事件的结构主要有3个版本：

* v1: Used in MySQL 3.23
* v3: Used in MySQL 4.0.2 though 4.1
* v4: Used in MySQL 5.0 and up

现在基本用的是v4版本。
```sql
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    |
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    |
|        +----------------------------+
|        | next_position    13 : 4    |
|        +----------------------------+
|        | flags            17 : 2    |
|        +----------------------------+
|        | extra_headers    19 : x-19 |
+=====================================+
| event  | fixed part        x : y    |
| data   +----------------------------+
|        | variable part              |
+=====================================+
```
上面用`offset : length`描述了各个属性的位置，如果事件头的长度是 x 字节，那么事件体的长度为 (event_length - x) 字节；设事件体中 fixed part 的长度为 y 字节，那么 variable part 的长度为 (event_length - (x + y)) 字节

### binlog-checksum
校验和功能是为了防止传输过程中发生差错，主从不一致。

关于binlog-checksum有三个参数，分别是
* `binlog_checksum`：默认值是CRC32，可以设置为`NONE`
* `master_verify_checksum`：主库校验event校验和，默认为0，在master thread进行dump的时候校验，在`SHOW BINLOG EVENTS`校验
* `slave_sql_verify_checksum`：从库校验event校验和，默认为1，当IO thread把event写入到relay log（从库读取到的binlog生成relay log）的时候校验。

mysqlbinlog可以加上`--verify-binlog-checksum`参数，打印有问题的sql。

如果校验失败会报错，可以用`pt-table-checksum`工具进行修正，关于更多，另行了解。

## SHOW BINLOG EVENTS
这个是mysql命令，也是用来阅读binlog。使用方式是`SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count]`，中括号都是可选项。

## GTID
可以在日志里看到`GTID`的字眼，GTID即Global Transaction ID，全局事务id，由`server_uuid:transaction_id`组成，是在mysql5.6引进的一个特性。

组复制插件**MGR**mysql官方的一个高可用插件，使用了PAXOS协议。在这种一致性协议中需要有一个全局增长的command index，GTID就承担了这个角色。

执行`show variables like '%gtid%';`，输出
```sql
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | OFF       |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | OFF       |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.02 sec)
```
`gtid_mode`其实是`OFF`状态。

关于更多，另行了解。

# 主从复制
主从复制有基于binlog和基于GTID两种方式。基于binlog的复制模式的基本流程是：

1. master事务提交，将记录变更写入binlog
2. slave的io进程连接master，从指定位置或从0开始请求日志
3. master返回日志给slave，并带上binlog名称和下一个binlog消费位置
4. slave接收到日志，将日志追加到relay log末端，并记录binlog名称和binlog消费位置，下次请求使带上。
5. slave的sql进程不断读relay log，执行sql。
6. 如果slave开启了binlog，又会将执行的sql变更记入binlog；如果slave又是其他slave的master，就会执行一样的逻辑。

在slave上执行，`show slave status\G`，部分输出如下
```sql
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 11.198.116.79
                  Master_User: replicator
                  Master_Port: 3018
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000831
          Read_Master_Log_Pos: 1151797
               Relay_Log_File: slave-relay.002490
                Relay_Log_Pos: 313
        Relay_Master_Log_File: mysql-bin.000831
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1151797
              Relay_Log_Space: 769
```
如果`Read_Master_Log_Pos`和`Exec_Master_Log_Pos`一致，表示从库已经追赶上主库。

在主库执行`show master status\G`，可以看到主库当前正在写入的binlog和位置。

# 参考
1. [《Event Structure》](https://dev.mysql.com/doc/internals/en/event-structure.html)
2. [《MySQL为什么默认隔离级别为可重复读？》](https://dominicpoi.com/2019/06/16/MySQL-1/)
3. [《10分钟学会Mysql之Binlog》](https://zhuanlan.zhihu.com/p/33504555)