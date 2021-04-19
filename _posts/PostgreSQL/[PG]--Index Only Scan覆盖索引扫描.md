---
title:[PG]--Index Only Scan覆盖索引扫描
date:2020-09-06
---



##### [PG]--Index Only Scan覆盖索引扫描

- [实验操作](实验操作)
- [特殊的覆盖索引扫描](特殊的覆盖索引扫描)
- [VM实验操作](VM实验操作)



### 实验操作

```
postgres=# create table t1 (id int,name varchar(10));
CREATE TABLE


postgres=# alter table t1 add constraint pk_t1 primary key (id);
ALTER TABLE


postgres=# \d
              List of relations
 Schema |   Name   |     Type      |  Owner   
--------+----------+---------------+----------
 public | t1       | table         | postgres
 public | test     | table         | postgres
 public | test_fdw | foreign table | postgres
(3 rows)


postgres=# \x
Expanded display is on.
postgres=# explain select * from t1 where id=1;
-[ RECORD 1 ]---------------------------------------------------------------
QUERY PLAN | Index Scan using pk_t1 on t1  (cost=0.15..8.17 rows=1 width=42)
-[ RECORD 2 ]---------------------------------------------------------------
QUERY PLAN |   Index Cond: (id = 1)



postgres=# explain select id from t1 where id=1;
-[ RECORD 1 ]-------------------------------------------------------------------
QUERY PLAN | Index Only Scan using pk_t1 on t1  (cost=0.15..8.17 rows=1 width=4)
-[ RECORD 2 ]-------------------------------------------------------------------
QUERY PLAN |   Index Cond: (id = 1)
```



### 特殊的覆盖索引扫描

![image.png](http://cdn.lifemini.cn/dbblog/20200906/65110596c51e491c8174b6e60e2298ec.png)

------

![image.png](http://cdn.lifemini.cn/dbblog/20200906/7c9045819c3f4b63ab88746cc0dee6d8.png)



### VM实验操作

```
postgres=# select proname from pg_proc where proname like '%filepath%';
       proname        
----------------------
 pg_relation_filepath
(1 row)


postgres=# select pg_relation_filepath('t1');
 pg_relation_filepath 
----------------------
 base/13593/65549
(1 row)


postgres=# exit
[postgres@localhost ~]$ cd /usr/local/pg12.2/data/base/13593/
Display all 306 possibilities? (y or n)
[postgres@localhost ~]$ cd /usr/local/pg12.2/data/base/13593/

[postgres@localhost 13593]$ ll |grep 6
6102   6104   6106   6110   6111   6112   6113   6117   65549  65552  

[postgres@localhost 13593]$ ll |grep 65549
-rw-------. 1 postgres postgres      0 Sep  6 03:27 65549


















postgres=# insert into t1 values (1,'n1');
INSERT 0 1
postgres=# insert into t1 values (2,'n2');
INSERT 0 1
postgres=# insert into t1 values (3,'n3');
INSERT 0 1
postgres=# insert into t1 values (4,'n4');
INSERT 0 1
postgres=# insert into t1 values (5,'n5');
INSERT 0 1
postgres=# insert into t1 values (6,'n6');
INSERT 0 1
postgres=# exit
[postgres@localhost 13593]$ ll |grep 65549
-rw-------. 1 postgres postgres   8192 Sep  6 03:49 65549
[postgres@localhost 13593]$ psql
psql (12.2)
Type "help" for help.

postgres=# insert into t1 values (7,'n7');
INSERT 0 1

postgres=# exit
[postgres@localhost 13593]$ vacuumdb --analyze --verbose --table 't1' postges
vacuumdb: error: could not connect to database postges: FATAL:  database "postges" does not exist
[postgres@localhost 13593]$ vacuumdb --analyze --verbose --table 't1' postgres
vacuumdb: vacuuming database "postgres"
INFO:  vacuuming "public.t1"
INFO:  index "pk_t1" now contains 7 row versions in 2 pages
DETAIL:  0 index row versions were removed.
0 index pages have been deleted, 0 are currently reusable.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
INFO:  "t1": found 0 removable, 7 nonremovable row versions in 1 out of 1 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 652
There were 0 unused item identifiers.
Skipped 0 pages due to buffer pins, 0 frozen pages.
0 pages are entirely empty.
CPU: user: 0.08 s, system: 0.07 s, elapsed: 0.16 s.
INFO:  analyzing "public.t1"
INFO:  "t1": scanned 1 of 1 pages, containing 7 live rows and 0 dead rows; 7 rows in sample, 7 estimated total rows
[postgres@localhost 13593]$ ll |grep 65549
-rw-------. 1 postgres postgres   8192 Sep  6 03:49 65549
-rw-------. 1 postgres postgres  24576 Sep  6 03:53 65549_fsm
-rw-------. 1 postgres postgres   8192 Sep  6 03:53 65549_vm
[postgres@localhost 13593]$ psql
psql (12.2)
Type "help" for help.

postgres=# explain select id from t1 where id=1;
                    QUERY PLAN                    
--------------------------------------------------
 Seq Scan on t1  (cost=0.00..1.09 rows=1 width=4)
   Filter: (id = 1)
(2 rows)

postgres=# explain select id from t1 where id=1;
                    QUERY PLAN                    
--------------------------------------------------
 Seq Scan on t1  (cost=0.00..1.09 rows=1 width=4)
   Filter: (id = 1)
(2 rows)

postgres=# explain select * from t1 where id=1;
                    QUERY PLAN                    
--------------------------------------------------
 Seq Scan on t1  (cost=0.00..1.09 rows=1 width=7)
   Filter: (id = 1)
(2 rows)

postgres=# insert into t1 values (8,'n8');
INSERT 0 1
postgres=# insert into t1 values (9,'n9');
INSERT 0 1
postgres=# explain select id from t1 where id=1;
                    QUERY PLAN                    
--------------------------------------------------
 Seq Scan on t1  (cost=0.00..1.09 rows=1 width=4)
   Filter: (id = 1)
(2 rows)

postgres=# explain select * from t1 where id=1;
                    QUERY PLAN                    
--------------------------------------------------
 Seq Scan on t1  (cost=0.00..1.09 rows=1 width=7)
   Filter: (id = 1)
(2 rows)
```