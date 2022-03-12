---
title: 深入学习 MySQL (一)：MySQL 体系结构
date: 2019-12-25 17:51:24
tags: MySQL
categories: 数据库
---

最初在使用 `MySQL` 数据库时，仅仅停留在增删改查的层次。缺乏对于数据库整体的认知。随着工作中接触到的需求越来越复杂，渐渐的认识到需要再次深入的去学习一下 `MySQL`。最近这几个月看了一些数据库相关的书籍，整理出了下面一些知识点。

<!-- more -->

## 数据库范式

数据库范式是为解决关系数据库中数据冗余、更新异常、插入异常、删除异常问题而引入的。简单的理解，数据库范式可以避免数据冗余，减少数据库的空间，并且减轻维护数据完整性的麻烦。

- 第一范式，强调属性的原子性约束，要求属性具有原子性，不可再分解。
- 第二范式，强调记录的唯一性约束，表必须有一个主键，并且没有包含在主键中的列必须完全依赖于主键，而不能只依赖于主键的一部分。
- 第三范式，强调属性冗余性的约束，即非主键列必须直接依赖于主键。

## 数据库的定义

在数据库体系中，**数据库** 与 **实例** 是两个不同的概念。很多开发人员对两者的区别不是很清楚。

- 数据库：物理操作系统文件或其他形式文件类型的集合
- 实例：MySQL 数据库后台线程以及一个共享的内存区域

一般来说，一个数据库会对应一个实例。但是在集群 RAC 共享数据文件时，一个数据库会对应多个实例。数据库是由一个个数据文件组成的，当我们需要进行 `CRUD` 等操作时，是不能通过粗暴的文件修改来完成的，需要通过数据库实例来完成对数据库的操作。

```Shell
# 查看数据文件
$ ls -al /usr/local/var/mysql
drwxr-xr-x   39 vincent  admin      1248 12 21 12:24 .
drwxrwxr-x    7 vincent  admin       224  6 17  2019 ..
drwxr-x---   19 vincent  admin       608  9 29 17:29 classicmodels
drwxr-x---   77 vincent  admin      2464  6 17  2019 mysql
drwxr-x---  108 vincent  admin      3456  6 17  2019 sys
drwxr-x---    5 vincent  admin       160 12 20 18:32 test
```

```Shell
# 查看数据库实例
$ ps -ef | grep mysqld
  501   374     1   0 12:24下午 ??         0:00.03 /bin/sh /usr/local/opt/mysql@5.7/bin/mysqld_safe --datadir=/usr/local/var/mysql
  501   474   374   0 12:24下午 ??         0:05.73 /usr/local/opt/mysql@5.7/bin/mysqld --basedir=/usr/local/opt/mysql@5.7 --datadir=/usr/local/var/mysql --plugin-dir=/usr/local/opt/mysql@5.7/lib/plugin --log-error=vincentdeMacBook-Pro.local.err --pid-file=vincentdeMacBook-Pro.local.pid
```

## 插件式存储引擎

`MySQL` 数据库区别与其他数据库的一个重要特点就是它的插件式表 `存储引擎`。存储引擎是基于表的，而不是数据库。这就意味着，可以在同一数据库的不同表中设置不同的存储引擎。这样做的好处是，每个存储引擎都有各自不同的特点，开发人员可以根据不同的业务逻辑建立不同的存储引擎表。

```
# 查看当前数据库对存储引擎的支持情况
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.01 sec)
```

在 `MySQL 5.5.8` 版本之前，默认的存储引擎是 `MyISAM`，`MyISAM` 存储引擎不支持事务，表锁设计，支持全文索引。
在 `MySQL 5.5.8` 版本之后，默认的存储引擎是 `InnoDB`，`InnoDB` 支持事务，行锁设计。两种存储引擎的区别在后续的文章会详细介绍。

## 文件系统

MySQL 的文件大致可以分为这几类：`参数文件` `日志文件` `Socket 文件` `PID 文件` `表结构文件` `数据文件`。

- `参数文件` 会在 MySQL 启动时被加载读取。文件中会定义数据文件的位置、各种内存大小的设置等。
- `日志文件` 记录了 MySQL 运行时的一些状态。有错误日志、慢查询日志、二进制日志等。
- `套接字文件` 当用 UNIX 套接字方式进行连接时锁需要的文件。
- `PID 文件` MySQL 实例进程 ID 文件
- `表结构文件` 用来存放 MySQL 表结构定义的文件
- `数据文件` 存储记录和索引数据。因为 MySQL 插件式存储引擎的原因，每个存储引擎都有自己的文件系统来存储记录和索引数据。所以每个存储引擎记录的数据文件格式并不一样。

## 启动与连接

MySQL 有四种启动方式，分别是 `mysqld` `mysqld_safe` `mysql.server` `mysqld_multi`。

- `mysqld` 是最常见的启动方式，启动后仅有一个 `mysqld` 进程。
- `mysqld_safe` 通过调用 `mysqld` 的方式启动实例，启动后会有两个进程，分别是 `mysqld` `mysqld_safe` 当实例出现错误时会自动重启。
- `mysql.server` 通过调用 `mysqld_safe` 的方式来启动实例
- `mysqld_multi` 多实例启动工具，用来管理多个 `mysqld` 进程，可以配置不同的实例参数

## 结构化查询语言

结构化查询语言，也就是我们通常所说的 `SQL` 语句。`SQL` 语句主要分为以下三个类别：

- DDL(Data Definition Languages) 数据定义语句，用于定义数据库、表、字段、索引、视图、触发器等。常用关键字为 `CREATE` `DROP` `ALTER` 等。
- DML(Data Manipulation Languages) 数据操纵语句，用于增删改查数据库记录，常用关键字为 `INSERT` `UPDATE` `SELECT` `DELETE` 等。
- DCL(Data Control Languages) 数据控制语句，用于控制用户对数据库、表、字段的访问权限和安全级别。常用关键字为 `GRANT` `REVOKE` 等。

## 参考连接

- [高性能 MySQL](https://book.douban.com/subject/23008813/ "高性能 MySQL")
- [MySQL 技术内幕：InnoDB 存储引擎](https://book.douban.com/subject/24708143/ "MySQL 技术内幕：InnoDB 存储引擎")
- [数据库索引设计与优化](https://book.douban.com/subject/26419771/ "数据库索引设计与优化")
- [MySQL 中文文档](https://www.mysqlzh.com/ "MySQL 中文文档")
- [MySQL 英文文档](https://dev.mysql.com/doc/refman/8.0/en/ "MySQL 英文文档")
- [mysqld_multi 多实例启动工具](http://wingyumin.com/2015/09/29/mysqld-multi多实例启动工具/ "mysqld_multi 多实例启动工具")
