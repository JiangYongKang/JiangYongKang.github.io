---
title: MySQL UNION 合并结果集
date: 2017-12-04 22:42:48
tags: MySQL
categories: 数据库
---

**UNION** 用于将多个 **SELECT** 语句的结果合并到结果集中。第一个 **SELECT** 语句的列名被用作返回结果的列列名。在每个 **SELECT** 语句的相应位置列出的选定列应该有相同的数据类型。

<!-- more -->

### 测试数据和表结构
```sql
mysql> desc books;
+-------------+---------------------+------+-----+---------+----------------+
| Field       | Type                | Null | Key | Default | Extra          |
+-------------+---------------------+------+-----+---------+----------------+
| id          | int(10) unsigned    | NO   | PRI | NULL    | auto_increment |
| name        | varchar(255)        | NO   |     | NULL    |                |
| create_time | datetime            | NO   |     | NULL    |                |
| is_hot      | tinyint(3) unsigned | NO   |     | 0       |                |
+-------------+---------------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
```
```sql
mysql> select * from books;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  1 | Head First Java                       | 2017-12-04 17:19:36 |      0 |
|  2 | Think in Java                         | 2017-12-02 17:19:36 |      0 |
|  3 | Clean Code                            | 2017-12-01 17:19:36 |      0 |
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  5 | JavaScript Definitive Guides          | 2017-12-06 17:19:36 |      0 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  7 | The C Programming Language            | 2017-12-08 17:19:36 |      0 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
+----+---------------------------------------+---------------------+--------+
8 rows in set (0.00 sec)
```

### 组合查询
**where** 和 **union** 语句在多数情况下可以实现相同的结果集。**where** 可以实现的语句，一定可以用 **union** 实现。而 **union** 可以实现的语句，**where** 却不一定能做到。因为，**union** 是可以针对多张表进行结果集的合并。
查询 `id = 1` 的记录
```sql
mysql> select * from books where id = 1;
+----+-----------------+---------------------+--------+
| id | name            | create_time         | is_hot |
+----+-----------------+---------------------+--------+
|  1 | Head First Java | 2017-12-04 17:19:36 |      0 |
+----+-----------------+---------------------+--------+
1 row in set (0.00 sec)
```
查询 `is_hot = 1` 的记录
```sql
mysql> select * from books where is_hot = 1;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
+----+---------------------------------------+---------------------+--------+
3 rows in set (0.00 sec)
```
使用 `union` 合并两个查询
```sql
mysql> select * from books where id = 1 union select * from books where is_hot = 1;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  1 | Head First Java                       | 2017-12-04 17:19:36 |      0 |
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
+----+---------------------------------------+---------------------+--------+
4 rows in set (0.00 sec)
```
当然，我们通过 `select * from books where id = 1 or is_hot = 1` 也可以做到，但是 **union** 是不限于表的。也就是说，我们可以从多个表中查询结果集出来，然后 **merge** 到第一个查询语句查询的结果集中（前提是字段数量和类型一致）。

### 重复项
默认情况下，我们单单使用 `union` 语句，会自动帮我们去除重复的项。但是，如果我们使用了 `union all` 语句，数据库不会帮我们自动去除重复项。
```sql
mysql> select * from books where id > 5 union select * from books where is_hot = 1;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  7 | The C Programming Language            | 2017-12-08 17:19:36 |      0 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
+----+---------------------------------------+---------------------+--------+
4 rows in set (0.00 sec)
```
```sql
mysql> select * from books where id > 5 union all select * from books where is_hot = 1;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  7 | The C Programming Language            | 2017-12-08 17:19:36 |      0 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
+----+---------------------------------------+---------------------+--------+
6 rows in set (0.00 sec)
```

### 关于排序
`union` 中的排序是比较坑的，即使我们在部分结果集这种写了 `order by` 语句。因为，`union` 会在合并结果集之后自动的进行排序。意味着在子结果集中的排序，排了也是白排。例如：
```sql
mysql> (select * from books where is_hot = 1 order by create_time desc)
    -> union
    -> (select * from books where is_hot = 0 order by create_time desc);
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
|  1 | Head First Java                       | 2017-12-04 17:19:36 |      0 |
|  2 | Think in Java                         | 2017-12-02 17:19:36 |      0 |
|  3 | Clean Code                            | 2017-12-01 17:19:36 |      0 |
|  5 | JavaScript Definitive Guides          | 2017-12-06 17:19:36 |      0 |
|  7 | The C Programming Language            | 2017-12-08 17:19:36 |      0 |
+----+---------------------------------------+---------------------+--------+
8 rows in set (0.00 sec)
```
从查询的意图来看，我们是想让 `is_hot = 1` 和 `is_hot = 0` 的结果集各自按 `create_time` 字段进行排序。但是结果出乎意外，并没有按我们设定的顺序进行。
其实，子结果集是进行了排序的，只不过 **union** 在合并结果集之后，会自动的对整个结果集进行排序。
假如我们现在有一个需求，`is_hot = 1` 的是热门书单，`is_hot = 0` 的是一般书单，然后按热门书单按 `create_time` 排序，普通书单也按 `create_time` 排序，返回一个集合。或者分页查询。我们该如何实现这个需求呢？查询语句如下：
```sql
mysql> (select * from books where is_hot = 1 order by create_time desc limit 7)
    -> union
    -> (select * from books where is_hot = 0 order by create_time desc limit 7);
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  7 | The C Programming Language            | 2017-12-08 17:19:36 |      0 |
|  5 | JavaScript Definitive Guides          | 2017-12-06 17:19:36 |      0 |
|  1 | Head First Java                       | 2017-12-04 17:19:36 |      0 |
|  2 | Think in Java                         | 2017-12-02 17:19:36 |      0 |
|  3 | Clean Code                            | 2017-12-01 17:19:36 |      0 |
+----+---------------------------------------+---------------------+--------+
8 rows in set (0.00 sec)
```
在每一个查询语句的后面都写上 `limit 7 (总条数)`。那么分页呢？
```sql
mysql> (select id, name, is_hot, create_time from books where is_hot = 1 order by create_time desc limit 7)
    -> union
    -> (select id, name, is_hot, create_time from books where is_hot = 0 order by create_time desc limit 7)
    -> limit 3 offset 0;
+----+---------------------------------------+--------+---------------------+
| id | name                                  | is_hot | create_time         |
+----+---------------------------------------+--------+---------------------+
|  8 | Pro Spring Boot                       |      1 | 2017-12-09 17:19:36 |
|  6 | Java 9 Data Structures and Algorithms |      1 | 2017-12-07 17:19:36 |
|  4 | The Ruby Programming                  |      1 | 2017-12-05 17:19:36 |
+----+---------------------------------------+--------+---------------------+
3 rows in set (0.00 sec)
```
```sql
mysql> (select id, name, is_hot, create_time from books where is_hot = 1 order by create_time desc limit 7)
    -> union
    -> (select id, name, is_hot, create_time from books where is_hot = 0 order by create_time desc limit 7)
    -> limit 3 offset 3;
+----+------------------------------+--------+---------------------+
| id | name                         | is_hot | create_time         |
+----+------------------------------+--------+---------------------+
|  7 | The C Programming Language   |      0 | 2017-12-08 17:19:36 |
|  5 | JavaScript Definitive Guides |      0 | 2017-12-06 17:19:36 |
|  1 | Head First Java              |      0 | 2017-12-04 17:19:36 |
+----+------------------------------+--------+---------------------+
3 rows in set (0.00 sec)
```

### 多表查询
创建一张杂志表 `magazines` 并插入两条数据。
```sql
mysql> desc magazines;
+-------------+---------------------+------+-----+---------+----------------+
| Field       | Type                | Null | Key | Default | Extra          |
+-------------+---------------------+------+-----+---------+----------------+
| id          | int(10) unsigned    | NO   | PRI | NULL    | auto_increment |
| name        | varchar(255)        | NO   |     | NULL    |                |
| create_time | datetime            | NO   |     | NULL    |                |
| is_hot      | tinyint(2) unsigned | NO   |     | 0       |                |
+-------------+---------------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
```
```sql
mysql> select * from magazines;
+----+--------------+---------------------+--------+
| id | name         | create_time         | is_hot |
+----+--------------+---------------------+--------+
|  1 | 男人装        | 2017-12-05 09:28:06 |      0 |
|  2 | 程序员        | 2017-12-01 09:28:06 |      1 |
+----+--------------+---------------------+--------+
3 rows in set (0.00 sec)
```
假如需求是从两张表或者多张表中查询出结果集，并且分页显示。那么这时候 `where` 很显然做不到，但是 `union` 却可以轻松做到。例如：查询 `books` 和 `magazines` 表中 `is_hot = 1` 的数据。
```sql
mysql> select * from books where is_hot = 1
    -> union
    -> select * from magazines where is_hot = 1;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
|  2 | 程序员                                 | 2017-12-01 09:28:06 |      1 |
+----+---------------------------------------+---------------------+--------+
4 rows in set (0.01 sec)
```

### 区分多表
联合多张表查询的时候，有时候会需要做到区分数据来自于哪一张表。比如，查询热门杂志和热门书籍，并且分别显示来自于哪一张表。
```sql
mysql> select *, 'books' as table_name from books where is_hot = 1
    -> union
    -> select *, 'magazines' as table_name from magazines where is_hot = 1;
+----+---------------------------------------+---------------------+--------+------------+
| id | name                                  | create_time         | is_hot | table_name |
+----+---------------------------------------+---------------------+--------+------------+
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 | books      |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 | books      |
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 | books      |
|  2 | 程序员                                 | 2017-12-01 09:28:06 |      1 | magazines  |
+----+---------------------------------------+---------------------+--------+------------+
4 rows in set (0.00 sec)
```
这样查询出来的结果集，封装成对象集合返回给前端，也方便前端对 `books` 或者 `magazines` 打不同的 `tag`。

### 具体应用场景
比如我们要对 `books` 表中的数据进行分页显示，要求热门书籍在上面，普通书籍在下面。热门书籍按 `create_time` 倒序，普通书籍也按 `create_time` 倒序。然后分页，每页显示 `5` 条数据。
第一页的数据：
```sql
mysql> (select * from books where is_hot = 1 order by create_time desc limit 8)
    -> union
    -> (select * from books where is_hot = 0 order by create_time desc limit 8)
    -> limit 5 offset 0;
+----+---------------------------------------+---------------------+--------+
| id | name                                  | create_time         | is_hot |
+----+---------------------------------------+---------------------+--------+
|  8 | Pro Spring Boot                       | 2017-12-09 17:19:36 |      1 |
|  6 | Java 9 Data Structures and Algorithms | 2017-12-07 17:19:36 |      1 |
|  4 | The Ruby Programming                  | 2017-12-05 17:19:36 |      1 |
|  7 | The C Programming Language            | 2017-12-08 17:19:36 |      0 |
|  5 | JavaScript Definitive Guides          | 2017-12-06 17:19:36 |      0 |
+----+---------------------------------------+---------------------+--------+
5 rows in set (0.01 sec)
```
第二页的数据：
```sql
mysql> (select * from books where is_hot = 1 order by create_time desc limit 8)
    -> union
    -> (select * from books where is_hot = 0 order by create_time desc limit 8)
    -> limit 5 offset 5;
+----+-----------------+---------------------+--------+
| id | name            | create_time         | is_hot |
+----+-----------------+---------------------+--------+
|  1 | Head First Java | 2017-12-04 17:19:36 |      0 |
|  2 | Think in Java   | 2017-12-02 17:19:36 |      0 |
|  3 | Clean Code      | 2017-12-01 17:19:36 |      0 |
+----+-----------------+---------------------+--------+
3 rows in set (0.00 sec)
```

### 参考资料
[https://dev.mysql.com/doc/refman/5.7/en/any-in-some-subqueries.html](https://dev.mysql.com/doc/refman/5.7/en/any-in-some-subqueries.html)
