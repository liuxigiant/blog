---
layout: post
title: "MySQL分区概念"
description: MySQL 分区概念
date: 2016-06-06 16:09:54 +0800
catalog: true
categories:
- MySQL
tags:
- MySQL Partitioning
---

**参考自官网：<https://dev.mysql.com/doc/refman/5.5/en/partitioning-overview.html>**  


## 概念  

- **分区：**使用特定的分区规则将一个表的数据分布到操作系统的不同目录和磁盘上。实际上，单个表的各个部分数据是作为分离的表存储在不同的位置的。  
- **分区规则：**完成分离表数据的分区方法，MySQL提供了[四种分区类型（range, list, hash, key）](https://dev.mysql.com/doc/refman/5.5/en/partitioning-types.html)。  
- **水平分区：**将一张表的不同行的数据分离到不同的物理分区，MySQL目前的分区方式就是水平分区。  
- **垂直分区：**将一张表的不同列的数据分离到不同的物理分区，MySQL 5.5不支持垂直分区。  


## MySQL存储引擎和分区

MySQL Server中的大部分存储引擎都支持分区表的创建（`MERGE`、`CSV`、`FEDERATED`存储引擎不支持），分区引擎与存储引擎运行在隔离的系统层面，并且可以相互交互。在MySQL 5.5，同一个分区表的所有分区必须是同一种存储引擎。  
例如：一个分区使用`MyISAM`存储引擎，而另一个分区使用`InnoDB`存储引擎。然而，MySQL Server也没有机制阻止不同的分区使用不同的存储引擎。  

>注意：  
>分区针对一张表的所有数据和所有索引生效。不能只对数据分区而不对索引分区，反之亦然。也不能只对表的部分数据分区。

## 分区的优势

- 分区能让一张表存储更多的数据  
- 当一些数据不再有较高的价值的时候，可以通过删除数据所在的分区，方便的删除数据。同样，需要增加一个特殊数据，也可以添加新的分区来存储  
- 查询优化：对于查询语句，可以根据`WHERE`条件过滤出需要查询的分区，而不用去查询其他数据不存在的分区（可以通过`EXPLAIN PARTITIONS SELECT`分析查询语句的执行计划）。对于已经存在的分区表，当前的分区方式若不能满足高频查询的需求，可以通过分区重建的方式来优化查询效率。  

**下面还有一些优化点：**  

- 对于`SUM()`、`COUNT()`等统计函数，分区查询能做到并行处理。例如：`SELECT salesperson_id, COUNT(orders) as order_total FROM sales GROUP BY salesperson_id;`这个查询语句，分区查询的并行处理优化的方式是，每个分区同时执行查询，最后对各分区结果做聚合。  
- 对于多个磁盘数据查询，实现更高的查询吞吐量

## 分区的限制

分区表也存在一些限制：  

- 算数运算符和逻辑运算符  
加减乘` + - *`这三个运算符是支持的，但是表达式结果必须是**整数**或者[NULL](https://dev.mysql.com/doc/refman/5.5/en/partitioning-handling-nulls.html)  
除法[DIV](https://dev.mysql.com/doc/refman/5.5/en/arithmetic-functions.html#operator_div)运算符也是支持的，但不支持`/`运算符  
位运算符`|  &  ^  <<  >>  ~ `是不支持的
- 锁表：在表上执行分区操作时候，会对表加上写锁，会影响对表的写入操作，不影响读操作。  
- 最大分区数：一张表的最大分区数为1024（表的存储引擎不是使用的NDB存储引擎，NDB存储引擎的最大分区数受MySQL Cluster版本、数据节点个数、和一些其他因素影响，详情可查看[NDB and user-defined partitioning](https://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-nodes-groups.html#mysql-cluster-nodes-groups-user-partitioning)）。1024包含分区及子分区。  
- 分区键（分区列）：对于分区列的限制在下篇单独说明

>以上只是简单的罗列出几点，详细情况可参考[官网文档](https://dev.mysql.com/doc/refman/5.5/en/partitioning-limitations.html)  

## 分区表管理

MySQL提供了[四种分区类型（range, list, hash, key）](https://dev.mysql.com/doc/refman/5.5/en/partitioning-types.html)。常用方式一般是根据时间进行分区，MySQL对`TO_DAYS()`, `YEAR()`, `TO_SECONDS()`这些处理时间的函数也进行了优化。下面给出一个简单示例：  

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
 
 
 /*
   增加分区
 */ 
ALTER TABLE t_test ADD PARTITION(PARTITION p1 VALUES LESS THAN (TO_DAYS('2016-07-01')));
```
上面的示例只是简单的创建分区表和给分区表增加分区，还要其他关于删除分区、重建分区等操作，可以参考官方文档：<https://dev.mysql.com/doc/refman/5.5/en/partitioning-management.html>

