---
title:[PG]-开启归档用pg_rman备份并基于时间点恢复
date:2020-11-09
---



把PG数据库变成归档，然后用pg_rman进行备份，做一个基于时间点的恢复案例。

### 数据库变成归档

```
cd $PGDATA
vi postgresql.conf

archive_mode = on 
archive_command = 'cp %p /home/postgres/arch/%f'
```



### 用pg_rman进行备份

```
[postgres@ pg1 ~]$ psql
psql (12.2)
Type "help" for help.

postgres=# Create database szp_db;
CREATE DATABASE
postgres=# \c szp_db
You are now connected to database "szp_db" as user "postgres".
szp_db=# Create table rman_test_table(id int,name varchar(20));
CREATE TABLE
szp_db=# Insert into rman_test_table values(1,'insert_1');
INSERT 0 1
szp_db=# Select now();
              now              
-------------------------------
 2020-09-28 14:13:02.560944-04
(1 row)
szp_db=# Insert into rman_test_table values(2,'insert_2');
INSERT 0 1
szp_db=# select * from rman_test_table;
 id |   name   
----+----------
  1 | insert_1
  2 | insert_2
(2 rows)
szp_db=# exit

[postgres@ pg1 ~]$ pg_rman backup --backup-mode=full -B /home/postgres/pg_rman_bk/ -C -P
INFO: copying database files
Processed 1315 of 1315 files, skipped 0
INFO: copying archived WAL files
Processed 23 of 23 files, skipped 0
INFO: backup complete
INFO: Please execute 'pg_rman validate' to verify the files are correctly copied.

[postgres@localhost ~]$ pg_rman show
=====================================================================
 StartTime           EndTime              Mode    Size   TLI  Status 
=====================================================================
2020-09-28 14:20:04  2020-09-28 14:20:33  FULL   307MB     1  DONE

[postgres@ pg1 ~]$ psql
psql (12.2)
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 szp_db    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            |
```



### 模拟误删数据库

```
postgres=CTc/postgres
(4 rows)
postgres=# drop database szp_db;
DROP DATABASE
postgres=# exit
[postgres@ pg1 ~]$ pg_ctl stop -m fast
waiting for server to shut down....... done
server stopped
```



### 基于时间点恢复

```
[postgres@ pg1 ~]$ pg_rman restore --recovery-target-time=' 2020-09-28 14:13:0';
INFO: backup "2020-09-28 14:20:04" is valid
…受篇幅限制，此处有省略…
INFO: restoring WAL files from backup "2020-09-28 14:20:04"
INFO: restoring online WAL files and server log files
INFO: add recovery related options to postgresql.conf
INFO: generating recovery.signal
INFO: restore complete
HINT: Recovery will start automatically when the PostgreSQL server is started.
[postgres@ pg1 ~]$ pg_ctl start
waiting for server to start....2020-09-28 14:24:30.396 EDT [27438] LOG:  starting PostgreSQL 12.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 9.3.1 20200408 (Red Hat 9.3.1-2), 64-bit
…受篇幅限制，此处有省略…
2020-09-28 14:24:31.659 EDT [27438] LOG:  database system is ready to accept read only connections
 done
server started
[postgres@pg1~]$ ps2020-09-28 14:24:33.453 EDT [27439] LOG:  restored log file "000000010000000000000005" from archive
ql2020-09-28 14:24:33.776 EDT [27439] LOG:  restored log file "000000010000000000000006" from archive
…受篇幅限制，此处有省略…
2020-09-28 14:24:38.389 EDT [27439] LOG:  recovery has paused
2020-09-28 14:24:38.389 EDT [27439] HINT:  Execute pg_wal_replay_resume() to continue.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 szp_db    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
postgres=# \c szp_db
You are now connected to database "szp_db" as user "postgres".
szp_db=# \d
              List of relations
 Schema |      Name       | Type  |  Owner   
--------+-----------------+-------+----------
 public | rman_test_table | table | postgres
(1 row)
szp_db=# select * from rman_test_table;
 id |   name   
----+----------
  1 | insert_1
(1 row)
[postgres@localhost ~]$ cd $PG_HOME/data/
[postgres@localhost data]$ ll |grep  recovery
-rw-rw-r--. 1 postgres postgres    45 Sep 28 14:24 recovery.signal
[postgres@localhost data]$ mv recovery.signal recovery.done
[postgres@localhost data]$ pg_ctl restart -m fast
waiting for server to shut down....2020-09-28 14:37:38.006 EDT [28006] LOG:  received fast shutdown request
…受篇幅限制，此处有省略…
2020-09-28 14:37:38.148 EDT [28107] LOG:  database system is ready to accept connections
 done
server started
```