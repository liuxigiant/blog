---
layout: post
title: "MySQL 对分区的支持"
description: MySQL Server查看对分区特性的支持
date: 2016-06-06 11:42:08 +0800
catalog: true
categories:
- MySQL
tags:
- MySQL Partitioning
---

**参考自官网：<https://dev.mysql.com/doc/refman/5.5/en/partitioning.html>**  


## BUG FIX

对分区表的了解源自一个bug，向历史表迁移数据时候报错，如下：  
SQL state [HY000]; error code [1526]; Table has no partition for value 736452;  
问题很简单，就是历史分区表没有新分区了，数据迁移不过去  

下面开始MySQL分区的学习之旅~~~  

## MySQL分区系统参数

在安装的MySQL服务端，可以通过以下方式查看，是否支持自定义分区特性：  

- mysql server 5.5版本可以通过 `SHOW VARIABLES` 命令查看  

```
mysql> SHOW VARIABLES LIKE '%partition%';

+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| have_partitioning | YES   |
+-------------------+-------+
1 row in set (0.00 sec)
```
>注意：
>have_partitioning变量在MySQL 5.6.1 版本已经被移除

- 可以使用`SHOW PLUGINS`命令查询，如下：  

```
mysql> SHOW PLUGINS;
+------------+----------+----------------+---------+---------+
| Name       | Status   | Type           | Library | License |
+------------+----------+----------------+---------+---------+
| binlog     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| partition  | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| ARCHIVE    | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| BLACKHOLE  | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| CSV        | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| FEDERATED  | DISABLED | STORAGE ENGINE | NULL    | GPL     |
| MEMORY     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| InnoDB     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| MRG_MYISAM | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| MyISAM     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| ndbcluster | DISABLED | STORAGE ENGINE | NULL    | GPL     |
+------------+----------+----------------+---------+---------+
11 rows in set (0.00 sec)
```
也可以查询`INFORMATION_SCHEMA.PLUGINS`表，会返回类似的结果：  

```
mysql> SELECT 
    ->     PLUGIN_NAME as Name, 
    ->     PLUGIN_VERSION as Version, 
    ->     PLUGIN_STATUS as Status 
    -> FROM INFORMATION_SCHEMA.PLUGINS 
    -> WHERE PLUGIN_TYPE='STORAGE ENGINE';
+--------------------+---------+--------+
| Name               | Version | Status |
+--------------------+---------+--------+
| binlog             | 1.0     | ACTIVE |
| CSV                | 1.0     | ACTIVE |
| MEMORY             | 1.0     | ACTIVE |
| MRG_MYISAM         | 1.0     | ACTIVE |
| MyISAM             | 1.0     | ACTIVE |
| PERFORMANCE_SCHEMA | 0.1     | ACTIVE |
| BLACKHOLE          | 1.0     | ACTIVE |
| ARCHIVE            | 3.0     | ACTIVE |
| InnoDB             | 5.6     | ACTIVE |
| partition          | 1.0     | ACTIVE |
+--------------------+---------+--------+
10 rows in set (0.00 sec)
```
若`partition`的`Status`列不是`ACTIVE`，则不支持分区。

## 分区特性支持

- ORACLE提供的MySQL 5.5社区版是包含分区的支持的；  
- 若从源码手动编译MySQL 5.5版本，需要配置` -DWITH_PARTITION_STORAGE_ENGINE `参数才能支持分区；  
- 若MySQL版本支持分区，则无需配置其他参数；  
- 若想关闭对分区的支持，在启动MySQL Server的时候，需要指定`--skip-partition `参数，这会是系统变量`have_partitioning`为`DISABLED`。当关闭了对分区的支持后，可以看见一些存在的分区表，也能删除分区表（不建议删除），但除此之外，分区表里的数据是不能操作和访问的。

## 获取分区信息

**参考自官网：<https://dev.mysql.com/doc/refman/5.5/en/partitioning-info.html>**  

- 使用`SHOW CREATE TABLE`语句查看建表语句中包含的分区条件  
  例如：SHOW CREATE TABLE table_name;

- 使用`SHOW TABLE STATUS`语句查看表是否分区  
  例如：SHOW TABLE STATUS FROM db_name;    查询结果中`Create_options`字段值为`partitioned`的表是分区表 

- 查询`information_schema.partitions`表 
  例如：SELECT * FROM  information_schema.partitions WHERE TABLE_SCHEMA='db_name' AND  TABLE_NAME ='table_name';  

- 分析查询语句的执行计划时候，使用`EXPLAIN PARTITIONS SELECT`，获取查询语句查询的分区信息  
  例如：  

``` sql
CREATE TABLE trb1 (id INT, name VARCHAR(50), purchased DATE)
    PARTITION BY RANGE(id)
    (
        PARTITION p0 VALUES LESS THAN (3),
        PARTITION p1 VALUES LESS THAN (7),
        PARTITION p2 VALUES LESS THAN (9),
        PARTITION p3 VALUES LESS THAN (11)
    );
INSERT INTO trb1 VALUES
    (1, 'desk organiser', '2003-10-15'),
    (2, 'CD player', '1993-11-05'),
    (3, 'TV set', '1996-03-10'),
    (4, 'bookcase', '1982-01-10'),
    (5, 'exercise bike', '2004-05-09'),
    (6, 'sofa', '1987-06-05'),
    (7, 'popcorn maker', '2001-11-22'),
    (8, 'aquarium', '1992-08-04'),
    (9, 'study desk', '1984-09-16'),
    (10, 'lava lamp', '1998-12-25');
```
查询语句的执行计划如下：  

```
mysql> EXPLAIN PARTITIONS SELECT * FROM trb1 WHERE id < 5\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: trb1
   partitions: p0,p1
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 10
        Extra: Using where
```
从查询语句的执行计划中可以看出，虽然trb1表分为四个分区，但是只查询了其中两个分区，这也是分区表的一个很重要的优化--[partition pruning](https://dev.mysql.com/doc/refman/5.5/en/partitioning-pruning.html)。