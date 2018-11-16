---
layout: post
title: "MySQL保存或更新 saveOrUpdate"
description: MySQL保存或更新
date: 2016-06-24 17:33:03 +0800
catalog: true
categories:
- MySQL
tags: 
- MySQL
---

## 1. 引子  
在项目开发过程中，有一些数据在写入时候，若已经存在，则覆盖即可。这样可以防止多次重复写入唯一键冲突报错。下面先给出两个MyBatis配置文件中使用saveOrUpdate的示例  

``` xml
<!-- 单条数据保存 -->
<insert id="saveOrUpdate" parameterType="TestVo">
	insert into table_name (
		col1,
		col2,
		col3
	)
	values (
		#{field1},
		#{field2},
		#{field3}
	)
	on duplicate key update
		col1 = #{field1},
		col2 = #{field2},
		col3 = #{field3}
</insert>  

<!-- 批量保存 -->
<insert id="batchSaveOrUpdate" parameterType="java.util.List">
	insert into table_name (
		col1,
		col2,
		col3
	)
	<foreach collection="list" item="item" index="index" separator=",">
		values (
			#{item.field1},
			#{item.field2},
			#{item.field3}
		)
	</foreach>
	on duplicate key update
		col1 = VALUES (col1),
		col2 = VALUES (col2),
		col3 = VALUES (col3)
</insert>
```  

>其实对于单行数据`on duplicate key update`也可以和批量数据保存一样使用`VALUES`表达式（`VALUES`指向新数据）。  

通过上面的例子初识MySQL `ON DUPLICATE KEY UPDATE`语法，下面继续学习~~

## 2. ON DUPLICATE KEY UPDATE 语法  
MySQL的`ON DUPLICATE KEY UPDATE`语法是指包含`ON DUPLICATE KEY UPDATE`子句的`INSERT`语句，当新增的这条语句在数据库中已经存在（已经存在是指这条数据包含的主键或者唯一键在数据库已经存在），则会更新数据库对应的老数据。  

下面两条sql语句就是等效的，其中table表中a是唯一键  

``` sql
INSERT INTO table (a,b,c) VALUES (1,2,3)
  ON DUPLICATE KEY UPDATE c=c+1;

UPDATE table SET c=c+1 WHERE a=1;
```
若在table表中，不仅仅存在a这个唯一键，b也是唯一键的情况下，以下两条语句就是等效的  

``` sql
INSERT INTO table (a,b,c) VALUES (1,2,3)
  ON DUPLICATE KEY UPDATE c=c+1;  

UPDATE table SET c=c+1 WHERE a=1 OR b=2 LIMIT 1;
```
>上面这条update语句的含义是：从表中取出**满足a=1或者b=2的一条数据**，进行更新操作。  

**下面重点了解以下几个问题：**  

### 2.1 多个唯一键 

>对于一张包含多个唯一键（多个唯一键指有多个键，而不是一个键中包含多个字段）的情况下，一定要注意多个唯一键是否会对应多条数据  


从上述第二个例子可以看出，`ON DUPLICATE KEY UPDATE`会根据`a=1或b=2`匹配出**一条数据**进行更新，当此时对应多条数据时候，这种更新操作就会有**不确定性**。（从另一个角度考虑，若多个唯一键都是一一对应，那么更新操作也不会有问题）  

### 2.2 影响行数返回值

>数据不存在，新增数据返回1 
>数据已存在，修改数据返回2
>数据已存在，但未变化返回0

**数据是否存在根据唯一键判断，数据是否修改根据`ON DUPLICATE KEY UPDATE`后的语句判断**  

下面是一个`ON DUPLICATE KEY UPDATE`返回值各种情况的简单实例：
``` sql
mysql> CREATE TABLE test1 (a INT PRIMARY KEY AUTO_INCREMENT , b INT, c INT);
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO test1(a, b ,c) VALUES (1, 1, 1);
Query OK, 1 row affected (0.00 sec)

mysql> select * from test1;
+---+------+------+
| a | b    | c    |
+---+------+------+
| 1 |    1 |    1 |
+---+------+------+
1 row in set (0.00 sec)

mysql> INSERT INTO test1(a, b ,c) VALUES (1, 1, 1) ON DUPLICATE KEY UPDATE c = c + 1;
Query OK, 2 rows affected (0.00 sec)

mysql> select * from test1;
+---+------+------+
| a | b    | c    |
+---+------+------+
| 1 |    1 |    2 |
+---+------+------+
1 row in set (0.00 sec)

mysql> INSERT INTO test1(a, b ,c) VALUES (2, 2, 2) ON DUPLICATE KEY UPDATE c = c + 1;
Query OK, 1 row affected (0.00 sec)

mysql> select * from test1;
+---+------+------+
| a | b    | c    |
+---+------+------+
| 1 |    1 |    2 |
| 2 |    2 |    2 |
+---+------+------+
2 rows in set (0.00 sec)

mysql> INSERT INTO test1(a, b ,c) VALUES (2, 2, 3) ON DUPLICATE KEY UPDATE c = VALUES(c);
Query OK, 2 rows affected (0.00 sec)

mysql> select * from test1;
+---+------+------+
| a | b    | c    |
+---+------+------+
| 1 |    1 |    2 |
| 2 |    2 |    3 |
+---+------+------+
2 rows in set (0.00 sec)
mysql> INSERT INTO test1(a, b ,c) VALUES (2, 2, 3) ON DUPLICATE KEY UPDATE c = VALUES(c);
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test1;
+---+------+------+
| a | b    | c    |
+---+------+------+
| 1 |    1 |    2 |
| 2 |    2 |    3 |
+---+------+------+
2 rows in set (0.00 sec)
```
注意返回值与新增、修改之间的关系  

### 2.3 新老数据引用 
>从上面的例子，和触发器做类比，在`ON DUPLICATE KEY UPDATE`子句后面，直接使用字段名，引用的是老数据；使用`VALUES`,引用的是要插入更新的新数据。（例如： `c=c+1`是在老数据的c字段上加1，`c=VALUES(c)`是拿新数据覆盖老数据）  

### 2.4 批量保存
批量保存使用`ON DUPLICATE KEY UPDATE`的场景，请回过头参照文章开始的示例中的第二个用法。  


**参考自官网：<http://dev.mysql.com/doc/refman/5.5/en/insert-on-duplicate.html>**  


