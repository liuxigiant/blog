---
layout: post
title: "MySQL分区字段的约束"
description: MySQL分区表分区字段与表唯一键字段之间的关系
date: 2016-06-07 19:41:41 +0800
catalog: true
categories:
- MySQL
tags:
- MySQL Partitioning
---


**参考自官网：<https://dev.mysql.com/doc/refman/5.5/en/partitioning-limitations-partitioning-keys-unique-keys.html>**  

``` sql
/*
  创建分区表
*/
CREATE TABLE `t_test` (
  `id` BIGINT(20) NOT NULL COMMENT 'id',
  `create_time` DATETIME NOT NULL COMMENT '创建时间',
  `user_name` VARCHAR(100) COMMENT '用户名',
   PRIMARY KEY (`id`,`create_time`)
) ENGINE=INNODB DEFAULT CHARSET=utf8
  PARTITION BY RANGE (TO_DAYS(`create_time`))
 (
    PARTITION p0 VALUES LESS THAN   (TO_DAYS('2016-07-01'))
 );
 
```


>从上面的示例中可以发现一个问题，为什么`create_time`字段需要作为联和主键中的一个呢？  

这引出了分区表的一个约束：**分区表中，分区表达式中的所有列，必须是表的`每一个`唯一键的一部分**。  
1. 主键也是唯一键  
2. 上述规则是当表存在唯一键的时候  
3. 所有唯一键都要匹配这个规则  

下面通过例子来理解以上规则：  

- 分区表达式包含的所有列不是唯一键的一部分
以下两种创建分区表的方式都是不合法的  

``` sql
CREATE TABLE t1 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col2)
)
PARTITION BY HASH(col3)
PARTITIONS 4;

CREATE TABLE t2 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1),
    UNIQUE KEY (col3)
)
PARTITION BY HASH(col1 + col3)
PARTITIONS 4;

mysql> CREATE TABLE t3 (
    ->     col1 INT NOT NULL,
    ->     col2 DATE NOT NULL,
    ->     col3 INT NOT NULL,
    ->     col4 INT NOT NULL,
    ->     UNIQUE KEY (col1, col2),
    ->     UNIQUE KEY (col3)
    -> )
    -> PARTITION BY HASH(col1 + col3)
    -> PARTITIONS 4;
ERROR 1491 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function  

```
  
下面给出正确的示例做对比：  
  

``` sql
CREATE TABLE t1 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col2, col3)
)
PARTITION BY HASH(col3)
PARTITIONS 4;

CREATE TABLE t2 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col3)
)
PARTITION BY HASH(col1 + col3)
PARTITIONS 4;

mysql> CREATE TABLE t3 (
    ->     col1 INT NOT NULL,
    ->     col2 DATE NOT NULL,
    ->     col3 INT NOT NULL,
    ->     col4 INT NOT NULL,
    ->     UNIQUE KEY (col1, col2, col3),
    ->     UNIQUE KEY (col3)
    -> )
    -> PARTITION BY HASH(col3)
    -> PARTITIONS 4;
Query OK, 0 rows affected (0.05 sec)  

```

- 分区表达中包含的列不是每一个唯一键的一部分，这条规则强调`每一个`

下面这种表不能分区，因为这张表的两个唯一键没有交集，没有分区列可以满足同时是这两个唯一键的一部分。  

``` sql
CREATE TABLE t4 (
    col1 INT NOT NULL,
    col2 INT NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col3),
    UNIQUE KEY (col2, col4)
);  

```

- 若分区表中没有唯一键（包括主键），那么则对分区字段没有限制，只要分区字段的值和分区类型兼容即可
  若给一个没有唯一键的分区表添加主键（通过 alter table语句），那么添加的唯一键与分区键之间同样有上述限制  

``` sql
mysql> CREATE TABLE t_no_pk (c1 INT, c2 INT)
    ->     PARTITION BY RANGE(c1) (
    ->         PARTITION p0 VALUES LESS THAN (10),
    ->         PARTITION p1 VALUES LESS THAN (20),
    ->         PARTITION p2 VALUES LESS THAN (30),
    ->         PARTITION p3 VALUES LESS THAN (40)
    ->     );
Query OK, 0 rows affected (0.12 sec)

#  possible PK
mysql> ALTER TABLE t_no_pk ADD PRIMARY KEY(c1);
Query OK, 0 rows affected (0.13 sec)
Records: 0  Duplicates: 0  Warnings: 0

# drop this PK
mysql> ALTER TABLE t_no_pk DROP PRIMARY KEY;
Query OK, 0 rows affected (0.10 sec)
Records: 0  Duplicates: 0  Warnings: 0

#  use another possible PK
mysql> ALTER TABLE t_no_pk ADD PRIMARY KEY(c1, c2);
Query OK, 0 rows affected (0.12 sec)
Records: 0  Duplicates: 0  Warnings: 0

# drop this PK
mysql> ALTER TABLE t_no_pk DROP PRIMARY KEY;
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

#  fails with error 1503
mysql> ALTER TABLE t_no_pk ADD PRIMARY KEY(c2);
ERROR 1503 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function  

```
