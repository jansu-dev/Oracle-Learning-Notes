---
title:[PG]--vaccum与freeze
date:2020-09-06
---



##### [PG]--vaccum急性冻结和惰性冻结

- [冻结](冻结)

- [clog](clog)

- [vaccum]( vaccum)



### 冻结

**急性冻结**

急性冻结属于I/O密集型操作，需要扫描所有数据块。
vm中记录哪些块是空闲的、那些是非空闲的，以此加快vaccum速度。

```
postgres=# select name,setting from pg_settings where name like '%freesze%';

 name | setting 
------+---------
(0 rows)
```

**惰性冻结**

```
postgres=# select datname,datfrozenxid from pg_database;
  datname  | datfrozenxid 
-----------+--------------
 postgres  |          480
 oracle    |          480
 template1 |          480
 template0 |          480
 d1        |          480
 u2_db     |          480
 u1_db     |          480
 test      |          480
 jantestdb |          480
(9 rows)
```

**二者区别**

惰性冻结时跳过的数据页，在急性冻结时将不会被被跳过。

当数据库中所有的表均被冻结过后，pg_class中relfrozenid才会改变。



### clog

10版本以前，ls -la -h data/clog
10版本以后，ls -la -h data/pg_xact

```
[postgres@pg1 pg_xact]$ pwd
/usr/local/pg12.2/data/pg_xact


[postgres@pg1 pg_xact]$ ll
total 8
-rw-------. 1 postgres postgres 8192 Aug 29 09:28 0000


[postgres@pg1 pg_xact]$ ps -ef|grep postgres |grep autovaccum
postgres   3079 123743  0 23:08 pts/0    00:00:00 grep --color=auto autovaccum
```



### vaccum

autovaccum

```
postgres=# show autovacuum_naptime;
 autovacuum_naptime 
--------------------
 1min
(1 row)

postgres=# show autovaccum_naptime;
ERROR:  unrecognized configuration parameter "autovaccum_naptime"
postgres=# show autovacuum_max_workers;
 autovacuum_max_workers 
------------------------
 3
(1 row)

postgres=# show autovacuum;
 autovacuum 
------------
 on
(1 row)
```

full vaccum

```
select count(*) as "number of pages",
pg_size_pretty(cast(avg(avail) as bigint)) as "Av freespace size",
rount(100 * avg(avail)/8192, 2) as "Av freespace ratio"
from pg_freespace('t1');
postgres=# show  log_autovacuum_min_duration;
 log_autovacuum_min_duration 
-----------------------------
 -1
(1 row)


[postgres@pg1 data]$ pwd
/usr/local/pg12.2/data
[postgres@pg1 data]$ cat postgresql.auto.conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.





1.当执行完整清理时，没有人可以访问（读/写）表。

2.最多会临时使用两倍于表的磁盘空间；因此在处理大表时，有必要检查剩余磁盘容量
```

**什么时候清理**

```
employe =1000行

--处罚事件
total number of obsolete recoreds = (0.2 * 1000) + 50 =250

--候选者
total number of insert/dletes/updates =（0.1 * 1000) + 50 = 150
```

查询表统计信息

```
postgres=# \x
Expanded display is on.
postgres=# select * from pg_stat_all_tables where relname='t1';
-[ RECORD 1 ]-------+-------
relid               | 40960
schemaname          | public
relname             | t1
seq_scan            | 1
seq_tup_read        | 512
idx_scan            | 
idx_tup_fetch       | 
n_tup_ins           | 0
n_tup_upd           | 0
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 0
n_dead_tup          | 0
n_mod_since_analyze | 0
last_vacuum         | 
last_autovacuum     | 
last_analyze        | 
last_autoanalyze    | 
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0
```