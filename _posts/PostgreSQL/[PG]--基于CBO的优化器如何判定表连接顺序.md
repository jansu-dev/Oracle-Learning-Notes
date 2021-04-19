---
title:[PG]--基于CBO的优化器如何判定表连接顺序
date:2020-09-14
---



##### [PG]--基于CBO的优化器如何判定表连接顺序

- [环境准备](环境准备)
- [优化器优化能力](优化器优化能力)
- [表连接顺序判定规则](表连接顺序判定规则)

##### 

### 环境准备

创建 tbl表

```
postgres=# CREATE TABLE tbl (id int PRIMARY KEY, data int);
CREATE TABLE
postgres=#  CREATE INDEX tbl_data_idx ON tbl (data);
CREATE INDEX
postgres=# INSERT INTO tbl SELECT generate_series(1,10000),generate_series(1,10000);
INSERT 0 10000
postgres=# ANALYZE;
ANALYZE
postgres=# \d tbl
                Table "public.tbl"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 data   | integer |           |          | 
Indexes:
    "tbl_pkey" PRIMARY KEY, btree (id)
    "tbl_data_idx" btree (data)

postgres=# select count(*) from tbl;
 count 
-------
 10000
(1 row)
```

创建tbl_a表

```
postgres=#  CREATE TABLE tbl_a (id int PRIMARY KEY, data int);
CREATE TABLE
postgres=# CREATE INDEX tbl_a_data_idx ON tbl_a (data);
CREATE INDEX
postgres=# INSERT INTO tbl_a SELECT generate_series(1,10000),generate_series(1,10000);
INSERT 0 10000
postgres=#  ANALYZE;
ANALYZE
postgres=# \d tbl_a
               Table "public.tbl_a"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 data   | integer |           |          | 
Indexes:
    "tbl_a_pkey" PRIMARY KEY, btree (id)
    "tbl_a_data_idx" btree (data)
```

创建tbl_b表

```
postgres=# CREATE TABLE tbl_b (id int PRIMARY KEY, data int);
CREATE TABLE
postgres=# CREATE INDEX tbl_b_data_idx ON tbl_b (data);
CREATE INDEX
postgres=# INSERT INTO tbl_b SELECT generate_series(1,10000),generate_series(1,10000);
INSERT 0 10000
postgres=# ANALYZE;
ANALYZE
postgres=# \d tbl_b
               Table "public.tbl_b"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 data   | integer |           |          | 
Indexes:
    "tbl_b_pkey" PRIMARY KEY, btree (id)
    "tbl_b_data_idx" btree (data)
```

创建tbl_c表

```
postgres=# CREATE TABLE tbl_c (id int PRIMARY KEY, data int);
CREATE TABLE
postgres=# CREATE INDEX tbl_c_data_idx ON tbl_c (data);
CREATE INDEX
postgres=# INSERT INTO tbl_c SELECT generate_series(1,10000),generate_series(1,10000);
INSERT 0 10000
postgres=#  ANALYZE;
ANALYZE
postgres=# \d tbl_c
               Table "public.tbl_c"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 data   | integer |           |          | 
Indexes:
    "tbl_c_pkey" PRIMARY KEY, btree (id)
    "tbl_c_data_idx" btree (data)
```

关闭hash连接模式和merge连接模式

```
postgres=# set enable_hashjoin=off;
SET
postgres=# set enable_mergejoin=off;
SET
postgres=# \x
Expanded display is on.
```



### 优化器优化能力

同一条SQL采用hashjoin连接方式总代价为182.52；

```
postgres=# set enable_hashjoin=on;
SET
postgres=# EXPLAIN analyze SELECT * FROM tbl_b AS b, tbl_a AS a WHERE a.id = b.id and a.id < 100;
-[ RECORD 1 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Hash Join  (cost=11.26..182.52 rows=99 width=16) (actual time=0.051..1.118 rows=99 loops=1)
-[ RECORD 2 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   Hash Cond: (b.id = a.id)
-[ RECORD 3 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Seq Scan on tbl_b b  (cost=0.00..145.00 rows=10000 width=8) (actual time=0.011..0.508 rows=10000 loops=1)
-[ RECORD 4 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Hash  (cost=10.02..10.02 rows=99 width=8) (actual time=0.030..0.031 rows=99 loops=1)
-[ RECORD 5 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Buckets: 1024  Batches: 1  Memory Usage: 12kB
-[ RECORD 6 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         ->  Index Scan using tbl_a_pkey on tbl_a a  (cost=0.29..10.02 rows=99 width=8) (actual time=0.005..0.020 rows=99 loops=1)
-[ RECORD 7 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |               Index Cond: (id < 100)
-[ RECORD 8 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.177 ms
-[ RECORD 9 ]---------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 1.147 ms
```

同一条SQL采用mergejoin连接方式总代价为354.54；

```
postgres=# set enable_hashjoin=off;
SET
postgres=# set enable_mergejoin=on;
SET
postgres=# set enable_nestloop=off;
SET
postgres=# EXPLAIN analyze SELECT * FROM tbl_b AS b, tbl_a AS a WHERE a.id = b.id and a.id < 100;
-[ RECORD 1 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Merge Join  (cost=0.57..354.54 rows=99 width=16) (actual time=0.014..0.084 rows=99 loops=1)
-[ RECORD 2 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   Merge Cond: (b.id = a.id)
-[ RECORD 3 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_b_pkey on tbl_b b  (cost=0.29..318.29 rows=10000 width=8) (actual time=0.007..0.019 rows=100 loops=1)
-[ RECORD 4 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_a_pkey on tbl_a a  (cost=0.29..10.02 rows=99 width=8) (actual time=0.004..0.018 rows=99 loops=1)
-[ RECORD 5 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id < 100)
-[ RECORD 6 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.216 ms
-[ RECORD 7 ]--------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 0.124 ms
```

同一条SQL采用nestloop连接方式总代价为339.97；

```
postgres=#  set enable_hashjoin=off;
SET
postgres=# set enable_mergejoin=off;
SET
postgres=# set enable_nestloop=on;
SET
postgres=# EXPLAIN analyze SELECT * FROM tbl_b AS b, tbl_a AS a WHERE a.id = b.id and a.id < 100;
-[ RECORD 1 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Nested Loop  (cost=0.57..339.97 rows=99 width=16) (actual time=0.011..0.110 rows=99 loops=1)
-[ RECORD 2 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_a_pkey on tbl_a a  (cost=0.29..10.02 rows=99 width=8) (actual time=0.004..0.014 rows=99 loops=1)
-[ RECORD 3 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id < 100)
-[ RECORD 4 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_b_pkey on tbl_b b  (cost=0.29..3.33 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=99)
-[ RECORD 5 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id = a.id)
-[ RECORD 6 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.104 ms
-[ RECORD 7 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 0.127 ms
```

三种表连接方式的代价分别以此为：

```
182.52 < 339.97 < 354.54
HashJoin < NestLoop < MergeJoin
```



### 表连接顺序判定规则

在不同结果集情况下，
PG优化器基于代价可以合理计算出内外表结果，
即：哪个适合作为内表，哪个适合作为外表。

```
postgres=# EXPLAIN analyze SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id;
-[ RECORD 1 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Nested Loop  (cost=0.29..3469.95 rows=10000 width=16) (actual time=0.021..10.087 rows=10000 loops=1)
-[ RECORD 2 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Seq Scan on tbl_a a  (cost=0.00..145.00 rows=10000 width=8) (actual time=0.010..0.686 rows=10000 loops=1)
-[ RECORD 3 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_b_pkey on tbl_b b  (cost=0.29..0.33 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=10000)
-[ RECORD 4 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id = a.id)
-[ RECORD 5 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.084 ms
-[ RECORD 6 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 10.489 ms
postgres=# EXPLAIN analyze SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id and a.id < 100;
-[ RECORD 1 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Nested Loop  (cost=0.57..339.97 rows=99 width=16) (actual time=0.009..0.092 rows=99 loops=1)
-[ RECORD 2 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_a_pkey on tbl_a a  (cost=0.29..10.02 rows=99 width=8) (actual time=0.003..0.011 rows=99 loops=1)
-[ RECORD 3 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id < 100)
-[ RECORD 4 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_b_pkey on tbl_b b  (cost=0.29..3.33 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=99)
-[ RECORD 5 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id = a.id)
-[ RECORD 6 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.086 ms
-[ RECORD 7 ]---------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 0.108 ms
```

在相同结果集情况下，
PG优化器按照FROM后面表的顺序，
取FROM后面第一个表作为嵌套循环表连接方式的外表。

```sql
postgres=# EXPLAIN analyze SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id;
-[ RECORD 1 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Nested Loop  (cost=0.29..3469.95 rows=10000 width=16) (actual time=0.016..8.817 rows=10000 loops=1)
-[ RECORD 2 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Seq Scan on tbl_a a  (cost=0.00..145.00 rows=10000 width=8) (actual time=0.006..0.571 rows=10000 loops=1)
-[ RECORD 3 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_b_pkey on tbl_b b  (cost=0.29..0.33 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=10000)
-[ RECORD 4 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id = a.id)
-[ RECORD 5 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.071 ms
-[ RECORD 6 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 9.151 ms
postgres=# EXPLAIN analyze SELECT * FROM tbl_b AS b, tbl_a AS a WHERE a.id = b.id;
-[ RECORD 1 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Nested Loop  (cost=0.29..3469.95 rows=10000 width=16) (actual time=0.019..11.703 rows=10000 loops=1)
-[ RECORD 2 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Seq Scan on tbl_b b  (cost=0.00..145.00 rows=10000 width=8) (actual time=0.008..0.767 rows=10000 loops=1)
-[ RECORD 3 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   ->  Index Scan using tbl_a_pkey on tbl_a a  (cost=0.29..0.33 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=10000)
-[ RECORD 4 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |         Index Cond: (id = b.id)
-[ RECORD 5 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Planning Time: 0.084 ms
-[ RECORD 6 ]----------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Execution Time: 12.163 ms
```