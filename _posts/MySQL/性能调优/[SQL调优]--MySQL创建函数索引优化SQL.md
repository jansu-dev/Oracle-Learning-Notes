---
title:[SQL调优]--MySQL创建函数索引优化SQL
date:2020-06-24
---



​		开发反应有一套MySQL宕机了，重启后在排查问题的过程中发现一条SQL执行的十分缓慢，最后通过创建函数索引的方式优化了该条SQL。

- [问题背景](问题背景)
- [问题现象](问题现象)
- [解决方案](解决方案)
- [案例总结](案例总结)



### 问题背景

使用pt-query-digest工具分析宕机当天的整天数据，
并结合开发应用日志的反馈，最终把问题定位到一个Event所执行的SQL。

```
pt-query-digest WIN-20200407-slow.log --since '2020-04-07 00:00:00' --until '2020-04-08 00:00:00' > analyse_20200407.log
```

t_XXX_general_indicator是一张50万行左右的基表，用于报表计算。

```
mysql> select count(1) from t_XXX_general_indicator;
+----------+
| count(1) |
+----------+
|   589499 |
+----------+
1 row in set (4.09 sec)
```



### 问题现象

问题SQL语句：
涉及企业信息已做处理

```
SELECT
    SUM( IFNULL( FIELD_27, 0 ) ) AS FIELD_27,
    SUM( IFNULL( FIELD_25, 0 ) ) AS FIELD_25,
    SUM( IFNULL( FIELD_26, 0 ) ) AS FIELD_26,
    SUM( IFNULL( FIELD_23, 0 ) ) AS FIELD_23,
    SUM( IFNULL( FIELD_24, 0 ) ) AS FIELD_24,
CASE

    WHEN SUM( IFNULL( FIELD_5, 0 ) ) > 0 THEN
    SUM( IFNULL( FIELD_12, 0 ) ) / SUM( FIELD_5 ) ELSE 0 
    END AS FIELD _18,
    SUM( IFNULL( FIELD_22, 0 ) ) AS FIELD_22,
    SUM( IFNULL( FIELD_15, 0 ) ) AS FIELD_15,
    SUM( IFNULL( FIELD_16, 0 ) ) AS FIELD_16,
    SUM( IFNULL( FIELD_7, 0 ) ) AS FIELD_7,
CASE

    WHEN SUM( IFNULL( FIELD_26, 0 ) ) > 0 THEN
    SUM( IFNULL( FIELD_24, 0 ) ) / SUM( FIELD_26 ) ELSE 0 
    END AS FIELD_29,
    SUM( IFNULL( FIELD_6, 0 ) ) AS FIELD_6,
CASE

    WHEN SUM( IF NULL ( FIELD_7, 0 ) ) > 0 THEN
    SUM( IFNULL( FIELD_26, 0 ) ) / SUM( FIELD_7 ) ELSE 0 
    END AS FIELD_28,
    SUM( IFNULL( FIELD_14, 0 ) ) AS FIELD_14,
    SUM( IFNULL( FIELD_5, 0 ) ) AS FI ELD_5,
    SUM( IFNULL( FIELD_4, 0 ) ) AS FIELD_4,
    SUM( IFNULL( FIELD_11, 0 ) ) AS FIELD_11,
    SUM( IFNULL( FIELD_10, 0 ) ) AS FIELD_10,
CASE

    WHEN SUM( IFNULL( FIELD_5, 0 ) ) > 0 T HEN SUM( IFNULL( FIELD_6, 0 ) ) / SUM( FIELD_5 ) ELSE 0 
    END AS FIELD_9,
CASE    
    WHEN SUM( IFNULL( FIELD_5, 0 ) ) > 0 THEN
    SUM( IFNULL( FIELD_7, 0 ) ) / SUM( FIELD_5 ) ELSE 0 
    END AS FIELD_8,
    SUM( IFNULL( FIELD_13, 0 ) ) AS FIELD_13,
    SUM( IFNULL( FIELD_12, 0 ) ) AS FIELD_12,
    MAX( IFNULL( FIELD_21, 0 ) ) AS FIELD_21,
    MAX( IFNULL( FIELD_20, 0 ) ) AS FIE LD_20,
    MAX( IFNULL( FIELD_19, 0 ) ) AS FIELD_19,
    MAX( IFNULL( FIELD_17, 0 ) ) AS FIELD_17 
FROM(
    SELECT
        SUBSTR( TIMERANGE, 1, 10 ) FIELD_1,
        T.SKILLID FIELD_2,
        T.SKIL LID FIELD_3,
        SUM( IBABANDONCALLS ) FIELD_4,
        SUM( ACDCALLS ) FIELD_5,
        SUM( ACDANSWERCALLS_30 ) FIELD_6,
        SUM( ACDANSWERCALLS ) FIELD_7,
        SUM( IBSHORTENABNCALLS ) F IELD_10,
        SUM( IBABANDONTIMELEN ) FIELD_11,
        SUM( IBWAITINGTIMELEN ) FIELD_12,
        SUM( INQUEUETIMELEN ) FIELD_13,
        SUM( IBTALKINGTIMELEN ) FIELD_14,
        SUM( IBACWTIMELE N ) FIELD_15,
        SUM( IBHANDLETIMELEN ) FIELD_16,
        MAX( MAXINQUEUETIMELEN ) FIELD_17,
        MAX( MAXWAITINGTIMELEN ) FIELD_19,
        MAX( IBMAXTALKINGTIMELEN ) FIELD_20,
        MAX( M AXABNINQUEUETIMELEN ) FIELD_21,
        SUM( IBACWNUM ) FIELD_22,
        SUM( ACDANSWERCALLS_30 ) - SUM( IBSHORTENABNCALLS ) FIELD_23,
        sum( T.STATISFYCALLS ) FIELD_24,
        sum( T.NOSTATISFYCALLS ) FIELD_25,
        sum( T.EVALUATECALLS ) FIELD_26,
        sum( T.ANSCALLS ) FIELD_27 
    FROM
        t_XXX_general_indicator T 
    WHERE
        T.SKILLID IN ('1111','2222','9970','9971','9980','9977','99771','99772','99773','99774','99775','99776','99777','99778','99779','99732','997311','9973121','9973122','9973123','9973131','9973132','9973133','9973134','997314','711191','711192','7112','7114','712','713','714','715','716','72','73','74','75','76','77','78','79','9979','301','302','303','304','305','306','307','308','309','310','311','312','314','315','320','321','322','323','324','325','326','9974' ) 
        AND SUBSTR( TIMERANGE, 1, 10 ) <= '2020-03-17' AND SUBSTR( TIMERANGE, 1, 10 ) >= '2020-03-17' 
    GROUP BY
        substr( TIMERANGE, 1, 10 ),
        T.SKILLID 
    ) TEMP_TAB 
    LIMIT 1;
```

查看执行计划，可以看到由于互动窗口计算原因，mysql优化器将该表查询当成了衍生表（DERIVED）。

```
mysql> explain SELECT SUM(IFNULL(FIELD_27,0)) AS ... ... GROUP BY substr(TIMERANGE, 1, 10), T.SKILLID) TEMP_TAB  limit 1;

+----+-------------+------------+------------+------+---------------------------------+------+---------+------+--------+----------+----------------------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra|
+----+-------------+------------+------------+------+---------------------------------+------+---------+------+--------+----------+----------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 220061 |   100.00 | NULL                         |
|  2 | DERIVED     | T          | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 440122 |    50.00 | Using where; Using temporary; Using filesort |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```



### 解决方案

最终，通过创建索引的方式优化。

```
mysql> create index ind_SKILLID on t_XXX_general_indicator(SKILLID);
Query OK, 0 rows affected (3.58 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

最终执行时间从10s以上，优化后至2.17s。

```
mysql> select count(1) as cnt  from (SELECT SUBSTR(TIMERANGE, 1, 10) FIELD_1 ...  ... AND SUBSTR(TIMERANGE, 1, 10) <= '2020-03-17' AND SUBSTR(TIMERANGE, 1, 10) >= '2020-03-17' GROUP BY substr(TIMERANGE, 1, 10), T.SKILLID )
t_1;
+-----+
| cnt |
+-----+
|  42 |
+-----+
1 row in set (2.17 sec)
```

首先，查看基表结构；

```
mysql> desc t_XXX_general_indicator;
+----------------------+---------------+------+-----+---------+-------+
| Field                | Type          | Null | Key | Default | Extra |
+----------------------+---------------+------+-----+---------+-------+
| TIMERANGE            | varchar(64)   | YES  |     | NULL    |       |
| SKILLID              | varchar(32)   | YES  |     | NULL    |       |
| IBABANDONCALLS       | decimal(54,0) | YES  |     | NULL    |       |
| ACDCALLS             | decimal(54,0) | YES  |     | NULL    |       |
| ACDANSWERCALLS_30    | decimal(54,0) | YES  |     | NULL    |       |
| ACDANSWERCALLS       | decimal(54,0) | YES  |     | NULL    |       |
| IBSHORTENABNCALLS    | decimal(54,0) | YES  |     | NULL    |       |
| IBABANDONTIMELEN     | decimal(54,0) | YES  |     | NULL    |       |
| IBWAITINGTIMELEN     | decimal(54,0) | YES  |     | NULL    |       |
| INQUEUETIMELEN       | decimal(54,0) | YES  |     | NULL    |       |
| IBTALKINGTIMELEN     | decimal(54,0) | YES  |     | NULL    |       |
| IBACWTIMELEN         | decimal(54,0) | YES  |     | NULL    |       |
| IBHANDLETIMELEN      | decimal(54,0) | YES  |     | NULL    |       |
| MAXINQUEUETIMELEN    | bigint(20)    | YES  |     | NULL    |       |
| MAXWAITINGTIMELEN    | bigint(20)    | YES  |     | NULL    |       |
| IBMAXTALKINGTIMELEN  | bigint(20)    | YES  |     | NULL    |       |
| MAXABNINQUEUETIMELEN | bigint(20)    | YES  |     | NULL    |       |
| IBACWNUM             | decimal(54,0) | YES  |     | NULL    |       |
| statisfyCalls        | decimal(45,0) | YES  |     | NULL    |       |
| noStatisfyCalls      | decimal(45,0) | YES  |     | NULL    |       |
| ansCalls             | decimal(45,0) | YES  |     | NULL    |       |
| evaluateCalls        | decimal(45,0) | YES  |     | NULL    |       |
+----------------------+---------------+------+-----+---------+-------+
22 rows in set (0.01 sec)
```

其次，增加伪劣；

```
mysql> alter table t_XXX_general_indicator add dba_subtimerange varchar(64) generated always as (SUBSTR(TIMERANGE, 1, 10));
Query OK, 0 rows affected (0.76 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc t_XXX_general_indicator;
+----------------------+---------------+------+-----+---------+-------------------+
| Field                | Type          | Null | Key | Default | Extra             |
+----------------------+---------------+------+-----+---------+-------------------+
| TIMERANGE            | varchar(64)   | YES  |     | NULL    |                   |
| SKILLID              | varchar(32)   | YES  |     | NULL    |                   |
| IBABANDONCALLS       | decimal(54,0) | YES  |     | NULL    |                   |
| ACDCALLS             | decimal(54,0) | YES  |     | NULL    |                   |
| ACDANSWERCALLS_30    | decimal(54,0) | YES  |     | NULL    |                   |
| ACDANSWERCALLS       | decimal(54,0) | YES  |     | NULL    |                   |
| IBSHORTENABNCALLS    | decimal(54,0) | YES  |     | NULL    |                   |
| IBABANDONTIMELEN     | decimal(54,0) | YES  |     | NULL    |                   |
| IBWAITINGTIMELEN     | decimal(54,0) | YES  |     | NULL    |                   |
| INQUEUETIMELEN       | decimal(54,0) | YES  |     | NULL    |                   |
| IBTALKINGTIMELEN     | decimal(54,0) | YES  |     | NULL    |                   |
| IBACWTIMELEN         | decimal(54,0) | YES  |     | NULL    |                   |
| IBHANDLETIMELEN      | decimal(54,0) | YES  |     | NULL    |                   |
| MAXINQUEUETIMELEN    | bigint(20)    | YES  |     | NULL    |                   |
| MAXWAITINGTIMELEN    | bigint(20)    | YES  |     | NULL    |                   |
| IBMAXTALKINGTIMELEN  | bigint(20)    | YES  |     | NULL    |                   |
| MAXABNINQUEUETIMELEN | bigint(20)    | YES  |     | NULL    |                   |
| IBACWNUM             | decimal(54,0) | YES  |     | NULL    |                   |
| statisfyCalls        | decimal(45,0) | YES  |     | NULL    |                   |
| noStatisfyCalls      | decimal(45,0) | YES  |     | NULL    |                   |
| ansCalls             | decimal(45,0) | YES  |     | NULL    |                   |
| evaluateCalls        | decimal(45,0) | YES  |     | NULL    |                   |
| dba_subtimerange     | varchar(64)   | YES  |     | NULL    | VIRTUAL GENERATED |
+----------------------+---------------+------+-----+---------+-------------------+
23 rows in set (0.00 sec)
```

最后，在位列上创建函数索引。

```
mysql> create index ind_subtimerange on  t_XXX_general_indicator(dba_subtimerange);
Query OK, 0 rows affected (4.19 sec)
Records: 0  Duplicates: 0  Warnings: 0



mysql> show index from  t_XXX_general_indicator;
+----------------------------------+------------+-----------------+--------------+------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table                            | Non_unique | Key_name         | Seq_in_index | Column_name      | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+----------------------------------+------------+------------------+--------------+------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t_XXX_general_indicator |          1 | ind_subtimerange |            1 | dba_subtimerange | A         |        1445 |     NULL | NULL   | YES  | BTREE      |         |               |
+----------------------------------+------------+------------------+--------------+------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)
```



### 案例总结

本次优化额核心是通过增加位列，在位列上创建索引的方式模拟Oracle的函数索引；
通过减少返回的结果集数量，达到优化SQL语句的目的。
核心语句如下：

```
alter table t_XXX_general_indicator add dba_subtimerange varchar(64) generated always as (SUBSTR(TIMERANGE, 1, 10));


create index ind_subtimerange on  t_report_XXX_indicator(dba_subtimerange);
```