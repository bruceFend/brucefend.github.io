---
layout: post
title: partitioning and foreign key
category: 技术
tags: MySQL 
keywords: MySQL 分区 外键
description: MySQL分区和外键的学习记录
---

在之前的开发经历中，从没接触过MySQL分区这一块儿，在最近的一次项目中，第一次使用到了。通常第一次总是伴随着很多坑，比如目标表里面有外键的话，是不能建分区的（使用默认的InnoDB引擎）；采用Hash方法时，被用来分区的列，需要是unique key（在项目中目标表是按日期来分，所以就跟id一起作为联合主键）。其实这些在MySQL的官方文档中都有写，只是平时没有用的时候也不会去看。。。

以下是看官方文档时候做的笔记：

//函数返回必须是非常数及非随机
> In the case of RANGE, LIST, and [LINEAR] HASH partitioning, the value of the partitioning column is passed to the partitioning function, which returns an integer value, and this function must be nonconstant and nonrandom.

//短期内都不计划支持列分区
> MySQL 5.7 does not support vertical partitioning, in which different columns of a table are assigned to different physical partitions. There are no plans at this time to introduce vertical partitioning into MySQL.

//MyISAM and InnoDB 引擎支持分区
> MySQL partitioning cannot be used with the MERGE, CSV, or FEDERATED storage engines.

//5.7 支持按分区搜索，加快查询速度
> MySQL 5.7 supports explicit partition selection for queries. For example, SELECT * FROM t PARTITION (p0,p1) WHERE c < 5 selects only those rows in partitions p0 and p1 that match the WHERE condition.

//建了分区表后，不要轻易改变SQL mode
> a change in the SQL mode at any time after the creation of partitioned tables  could easily lead to corruption or loss of data.  *it is strongly recommended that you never change the server SQL mode after creating partitioned tables.*

### 一、[Partitioning Types](https://dev.mysql.com/doc/refman/5.7/en/partitioning-types.html)

- [RANGE partitioning.](https://dev.mysql.com/doc/refman/5.7/en/partitioning-range.html) 
- [LIST partitioning. ](https://dev.mysql.com/doc/refman/5.7/en/partitioning-list.html)
- [HASH partitioning.](https://dev.mysql.com/doc/refman/5.7/en/partitioning-hash.html)
- [KEY partitioning.](https://dev.mysql.com/doc/refman/5.7/en/partitioning-key.html)

#### 1.Range partitioning

//分区范围需要连续且不能重合
> Ranges should be contiguous but not overlapping, and are defined using the `VALUES LESS THAN` operator.

//分区名大小写不敏感
> Names of partitions generally follow the rules governing other MySQL identifiers, such as those for tables and databases. However, partition names are ***not case-sensitive***.

//指定分区数量时：不能是表达式,需要是非0正整数
> When you specify the number of partitions for the table, this must be expressed as a positive, nonzero integer literal with no leading zeros, and ***may not be an expression*** such as 0.8E+01 or 6-2, even if it evaluates to an integer value. Decimal fractions are not permitted.

//适用条件：
> ***Range partitioning*** is particularly useful when one or more of the following conditions is true:
> - 【快速删除】You want or need to delete “old” data. Better to use `ALTER TABLE employees DROP PARTITION p0;` than `DELETE FROM employees WHERE YEAR(separated) <= 1990;`.
>
> - 【该列是日期类型或者以某种序列增大】You want to use a column containing date or time values, or containing values arising from some other series.
>
> - 【经常依据某一列进行查询】You frequently run queries that depend directly on the column used for partitioning the table.

#### 2.List partitioning

//与range分区的最大区别就是一个是值在list中就行，一个需要在连续的范围内
> in list partitioning, each partition is defined and selected based on the membership of a column value in one of *a set of value lists*, rather than in one of *a set of contiguous ranges of values*. 

//必须把所有的情况都列举出来，否则插入不匹配的数据时会出错
> Unlike the case with RANGE partitioning, there is no “catch-all” such as MAXVALUE; 

//list partition的好处/作用：可以使用非整型列分区，支持多列作为分区条件【range partitioning也可以】
> enables you to use columns of types *other than integer types* for partitioning columns, as well as to use *multiple columns* as partitioning keys.

//不能像range那样用`values less than`的方法，
> You cannot use `VALUES LESS THAN` with PARTITION BY LIST use `VALUES IN`.

#### 3.Hash partitioning

//为了保证分配均匀
> primarily to ensure an even distribution of data

//只需要确定方法和数量（默认1个分区）
> you need only specify a column value or expression based on a column value to be hashed and the number of partitions

//如果表中含有唯一key约束，hash分区表达式中涉及到的列，必须属于唯一key约束的列
> if a table has any unique keys, every column used in the partitioning expression for this table must be part of every unique key, including the primary key. 
【ERROR 1491 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function】

//默认用取模的方式决定数据的分区位置
> for a given expression `expr` （`expr` is an expression that returns an integer）, the partition in which the record is stored is partition number `N`, where `N = MOD(expr, num)`. 

//用于分区的列，列值与分区表达式的关系越接近线性越适合hash分区
>  the more closely the graph of the column value versus the value of the expression follows a straight line as traced by the equation `y=cx` where `c` is some nonzero constant, the better the expression is suited to hashing.

//多列的hash分区不被推荐（range、list和key都支持）
> In theory, pruning is also possible for expressions involving more than one column value, but determining which of such expressions are suitable can be quite difficult and time-consuming. For this reason, the use of hashing expressions involving multiple columns is ***not particularly recommended***.

//【重要？】线性hash分区，当分区数量为2的倍数时，普通hash分区和linear hash分区，无差别
> MySQL also supports linear hashing, which differs from regular hashing in that linear hashing utilizes a linear powers-of-two algorithm whereas regular hashing employs the modulus of the hashing function's value.

#### 4.KEY partitioning

//与hash有点相似，但key分区方式用的是自己的hash函数，InnoDB采用的是MD5的方式。
> Partitioning by key is similar to partitioning by hash,the hashing function for key partitioning is supplied by the MySQL server.

//与hash分区方式的主要区别：（支持多列，不限制value必须是数值类型）
- KEY is used rather than HASH.
- KEY takes only a list of zero or more column names. 
- columns used for partitioning by KEY are not restricted to integer or NULL values.

----

### 二、[Restrictions and Limitations](https://dev.mysql.com/doc/refman/5.7/en/partitioning-limitations.html)

- All columns used in the table's partitioning expression must be part of every unique key that the table may have, including any primary key. 

> cannot be partitioned:
```MySQL
CREATE TABLE tnp (
    id INT NOT NULL AUTO_INCREMENT,
    ref BIGINT NOT NULL,
    name VARCHAR(255),
    PRIMARY KEY pk (id),
    UNIQUE KEY uk (name)
);
```
> Because the keys pk and uk have no columns in common, there are no columns available for use in a partitioning expression. 

- The bit operators |, &, ^, <<, >>, and ~ are not permitted in partitioning expressions.
- **Maximum number of partitions.**  The maximum possible number of partitions for a given table not using the NDB storage engine is 8192. This number includes subpartitions.
- **Query cache not supported.**  The query cache is not supported for partitioned tables, and is automatically disabled for queries involving partitioned tables. The query cache cannot be enabled for such queries.
- **Foreign keys not supported for partitioned InnoDB tables.**【InnoDB引擎的分区表不支持外键】
- **Temporary tables.  Temporary tables cannot be partitioned. (Bug #17497)**【临时表和log表不支持分区】
- Issues with subpartitions.  Subpartitions must use **HASH** or **KEY** partitioning. Only **RANGE** and **LIST** partitions may be subpartitioned; **HASH** and **KEY** partitions cannot be subpartitioned.【range和list方式才支持子分区，子分区类型只能是hash或者key】
- FULLTEXT indexes.  Partitioned tables do not support FULLTEXT indexes or searches【不支持全文索引】
- Subqueries.  A partitioning key may not be a subquery, even if that subquery resolves to an integer value or NULL.【不能作为子查询】
- Each partition must have the same number of subpartitions.

// MyISAM重分区时，
> For a partitioned MyISAM table, MySQL uses 2 file descriptors for each partition, for each such table that is open. Assume that you wish to repartition t so that it has 101 partitions, using the statement shown here:
```
ALTER TABLE t PARTITION BY KEY (c1) PARTITIONS 101;
```
> To process this ALTER TABLE statement, MySQL uses 402 file descriptors—that is, two for each of the 100 original partitions, plus two for each of the 101 new partitions.

---

三、

查看MySQL partition的情况：
```
mysql> SELECT 
       PARTITION_ORDINAL_POSITION, 
       TABLE_ROWS, PARTITION_METHOD
    FROM information_schema.PARTITIONS 
    WHERE TABLE_SCHEMA = 'db_name' AND TABLE_NAME = 'tbl_name';
```

| PARTITION_ORDINAL_POSITION | TABLE_ROWS | PARTITION_METHOD |
|----------------------------|------------|------------------|
|                          1 |          2 | HASH             |
|                          2 |          3 | HASH             |


---

插播：

MySQL建立外键约束，如果数据类型不一致，会出错（error 1215）
> Corresponding columns in the foreign key and the referenced key must have similar data types. The size and sign of integer types must be the same. The length of string types need not be the same. For nonbinary (character) string columns, the character set and collation must be the same.[1]

- [1] https://dev.mysql.com/doc/refman/5.7/en/create-table-foreign-keys.html
