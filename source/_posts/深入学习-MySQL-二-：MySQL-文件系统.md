---
title: 深入学习 MySQL (二)：MySQL 文件系统
date: 2019-12-25 17:51:35
tags: MySQL
categories: 数据库
---

本章将分析构成 MySQL 数据库和 InnoDB 引擎的各种类型的文件。从物理角度来看，这些文件大致分为以下几类：

<!-- more -->

- 套接字文件：当客户端使用 UNIX 域套接字连接时所需要的文件
- 配置文件：启动时加载，配置 MySQL 和 存储引擎的初始设置
- 表结构文件：用来存放用户定义的数据表结构
- 表空间文件：存放表中的记录、索引等数据
- 日志文件：记录 MySQL 在运行过程中发生的一些事件

### 套接字文件

在 UNIX 系统下本机连接 MySQL 实例可以采用套接字文件的方式，这种方式需要套接字文件。查看本机套接字文件所在路径：

```Text
mysql> show variables like 'socket';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| socket        | /tmp/mysql.sock |
+---------------+-----------------+
1 row in set (0.00 sec)
```

### 配置文件

MySQL 实例在启动时，会先去读取配置文件，用来查找数据库各种文件所在的位置以及指定某些初始化参数。MySQL 会按照一定的顺序去查找配置文件，并且以最后一个查找到的配置文件为准。这点，与 Linux Shell 配置文件的优先级类似。

```Shell
# 查看配置文件的查找顺序与优先级
$ mysql --help
...
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/local/etc/my.cnf ~/.my.cnf
...
```

当然，这默认情况下这些文件中都是空空如也，每一个配置项无须我们一一配置，MySQL 都已经帮我们设置好了默认值，我们只需要配置自己需要作出修改的参数。如何查看所有的配置项以及默认值？官方推荐的做法是使用 `show variables` 命令，如下：

```Text
mysql> show variables;
+--------------------------------------------+------------------------------+
| Variable_name                              | Value                        |
+--------------------------------------------+------------------------------+
| activate_all_roles_on_login                | OFF                          |
| auto_generate_certs                        | ON                           |
| auto_increment_increment                   | 1                            |
| auto_increment_offset                      | 1                            |
| autocommit                                 | ON                           |
| automatic_sp_privileges                    | ON                           |
| avoid_temporal_upgrade                     | OFF                          |
| back_log                                   | 151                          |
| basedir                                    | /usr/                        |
| ...                                        |                              |
+--------------------------------------------+------------------------------+
```

上面截取了部分默认参数配置，如果需要对其进行修改，可以在配置文件中进行修改。但是 MySQL 实例需要重启才能生效，那么有没有办法可以在实例运行中进行修改参数呢？答案是可以的，但是仅限于部分参数。

MySQL 中的参数类型可以分为两类：

- 动态参数：可以在实例运行时进行修改，即时生效
- 静态参数：仅可以在配置文件中进行设置，实例需要重启才能生效

例如修改缓存区大小，这个就是属于动态参数：

```Text
mysql> set session read_buffer_size = 524288;  -- 增大缓冲区
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'read_buffer_size'; -- 查看是否设置成功
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| read_buffer_size | 524288 |
+------------------+--------+
1 row in set (0.00 sec)
```

如果试图在修改一个静态参数，那么可能得到以下报错：

```Text
mysql> set session basedir = '/';
ERROR 1238 (HY000): Variable 'basedir' is a read only variable
```

### 表结构文件

MySQL 的数据存储是按照表进行的，每个表都与之对应的表结构文件，文件中记录了表中的字段、类型、索引等信息。表结构文件的后缀名为 `.frm`。表结构文件与创建表时选择的存储引擎无关，选择了任意类型的存储引擎，都会在创建表时生成 `.frm` 文件。

### 表空间文件

InnoDB 存储引擎提供了两种表空间的存储方式：

- 共享表空间
- 独占表空间

在共享表空间的模式下，整个数据库的表数据和索引都被存放在同一个文件中，默认位于 `$datadir/ibdata1`。
在独占表空间的模式下，MySQL 会为每一张表生成一个独立的表空间，后缀名为 `.ibd`, 默认位于 `$datadir/$db_name/$table_name.ibd`。可以通过设置 `innodb_file_per_table=ON` 来开启独占表空间模式。

### 日志文件

日志记录 MySQL 在运行过程中发生的一些事件，查看分析日志文件可以对实例的运行状态进行诊断，从而可以更好的管理数据库。日志文件大致可以分为以下几类：

- 错误日志
- 查询日志
- 慢查询日志
- 二进制日志

**错误日志** 对 MySQL 的启动、运行、关闭过程进行了记录。当数据库实例发生异常时，我们首先应该检查的就是错误日志。查看日志文件的位置：

```Text
mysql> show variables like 'log_error';
+---------------+----------------------------------+
| Variable_name | Value                            |
+---------------+----------------------------------+
| log_error     | ./vincentdeMacBook-Pro.local.err |
+---------------+----------------------------------+
1 row in set (0.00 sec)
```

这是一个相对路径，默认的文件名是 `主机名.local.err`。查看一下错误日志的文件内容：

```Text
$ less /usr/local/var/mysql/vincentdeMacBook-Pro.local.err
2019-06-17T06:10:22.6NZ mysqld_safe Logging to '/usr/local/var/mysql/vincentdeMacBook-Pro.local.err'.
2019-06-17T06:10:22.6NZ mysqld_safe Starting mysqld daemon with databases from /usr/local/var/mysql
2019-06-17T06:10:22.983791Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2019-06-17T06:10:22.984135Z 0 [Note] --secure-file-priv is set to NULL. Operations related to importing and exporting data are disabled
2019-06-17T06:10:22.984193Z 0 [Note] /usr/local/opt/mysql@5.7/bin/mysqld (mysqld 5.7.25) starting as process 26665 ...
2019-06-17T06:10:22.987211Z 0 [Warning] Setting lower_case_table_names=2 because file system for /usr/local/var/mysql/ is case insensitive
```

**查询日志** 记录了所有对 MySQL 数据库请求的信息，无论这些请求有没有正确执行。查看日志文件的默认配置：

```Text
-- 查询日志是否开启
mysql> show variables like 'general_log';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| general_log   | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

```Text
-- 查询日志存放位置
mysql> show variables like 'general_log_file';
+------------------+-----------------------------------------------+
| Variable_name    | Value                                         |
+------------------+-----------------------------------------------+
| general_log_file | /usr/local/var/mysql/vincentdeMacBook-Pro.log |
+------------------+-----------------------------------------------+
1 row in set (0.01 sec)
```

```Text
-- 日志文件的存储方式
mysql> show variables like 'log_output';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set (0.01 sec)
```

**慢查询日志** 用来定位查询时间过长的 SQL 语句。默认配置下并不会开启，开启后查询时间超过 10s 的 SQL 语句都会被记录在慢查询日志中。查看慢查询日志的配置：

```Text
-- 慢查询日志是否开启
mysql> show variables like 'slow_query_log';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| slow_query_log | ON    |
+----------------+-------+
1 row in set (0.00 sec)
```

```Text
-- 慢查询日志阈值
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.01 sec)
```

```Text
-- 慢查询日志存储位置
mysql> show variables like 'slow_query_log_file';
+---------------------+----------------------------------------------------+
| Variable_name       | Value                                              |
+---------------------+----------------------------------------------------+
| slow_query_log_file | /usr/local/var/mysql/vincentdeMacBook-Pro-slow.log |
+---------------------+----------------------------------------------------+
1 row in set (0.00 sec)
```

```Text
-- 是否记录未使用索引的查询语句
mysql> show variables like 'log_queries_not_using_indexes';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | OFF   |
+-------------------------------+-------+
1 row in set (0.00 sec)
```

**二进制日志** 记录了对 MySQL 数据库执行更改的所有操作。但是不包括 `SELECT` `SHOW` 这类读的操作。二进制日志的主要作用是 `数据恢复` 与 `数据同步`。默认配置下二进制日志是关闭的，在配置文件中添加 `log-bin` 可以开启。

```Text
[mysqld]
log-bin=binlog
```

二进制日志默认的位置在 `datadir` 文件夹内，如：

```Text
$ ls -al /usr/local/var/mysql
total 389160
drwxr-xr-x   32 vincent  admin      1024 12 25 14:30 .
drwxrwxr-x    7 vincent  admin       224  6 17  2019 ..
-rw-r-----    1 vincent  admin        56  6 17  2019 auto.cnf
-rw-r-----    1 vincent  admin       728 12 25 14:35 bin_log.000001 # 二进制日志文件
-rw-r-----    1 vincent  admin        17 12 25 14:30 bin_log.index  # 二进制日志索引文件
```

以默认的 InnoDB 为例，所有未提交的事务产生的二进制日志文件会先写入到内存中，什么时间将内存中的二进制日志文件刷新到磁盘由 `sync_binlog` 参数的配置相关。配置 `sync_binlog=N` 则表示每缓存 N 次之后写入磁盘，即每提交 N 次事务后写入磁盘。

查看二进制日志文件的内容，必须使用 MySQL 提供的 `mysqlbinlog` 工具去查看

```Text
$ mysqlbinlog -v --base64-output=decode-rows /usr/local/var/mysql/bin_log.000001
#191225 17:28:16 server id 1  end_log_pos 1689 CRC32 0x5b9f6991 	Update_rows: table id 239 flags: STMT_END_F
### UPDATE `classicmodels`.`customers`
### WHERE
###   @1=103
###   @2='Atelier graphique'
###   @3=''
###   @4='Carine '
###   @5='40.32.2555'
###   @6='54, rue Royale'
###   @7=NULL
###   @8='Nantes'
###   @9=NULL
###   @10='44000'
###   @11='France'
###   @12=1370
###   @13=21000.00
### SET
###   @1=103
###   @2='Atelier graphique'
###   @3='Schmitt'
###   @4='Carine '
###   @5='40.32.2555'
###   @6='54, rue Royale'
###   @7=NULL
###   @8='Nantes'
###   @9=NULL
###   @10='44000'
###   @11='France'
###   @12=1370
###   @13=21000.00
# at 1689
#191225 17:28:16 server id 1  end_log_pos 1720 CRC32 0xd13f3d52 	Xid = 117
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

### 参考文章

- [InnoDB 文件系统之文件物理结构](http://mysql.taobao.org/monthly/2016/02/01/ "InnoDB 文件系统之文件物理结构")
- [InnoDB 文件系统之 IO 系统和内存管理 ](http://mysql.taobao.org/monthly/2016/02/02/ "InnoDB 文件系统之 IO 系统和内存管理 ")
- [InnoDB 的磁盘文件及落盘机制](https://juejin.im/post/5b94884af265da0ae74f60d3 "InnoDB 的磁盘文件及落盘机制")
- [MySQL Binlog 介绍](https://juejin.im/post/5c85cecb6fb9a049fb447c82 "MySQL Binlog 介绍")
