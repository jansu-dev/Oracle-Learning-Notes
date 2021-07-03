---
title:[PG]-逻辑备份与物理备份
date:2020-09-22
---



##### [PG]-逻辑备份与物理备份

- [备份方式区别](备份方式区别)
- [逻辑备份](逻辑备份)
- [本地外部表](本地外部表)
- [热备](热备)



### 备份方式区别

逻辑备份适合小量数据，否则可能再备份导出时，可能需要耗费与物理备份相比更多的时间。



### 逻辑备份

**pg_dump**
最多只能导出一个数据库，不会导出角色和表空间相关的信息。
导出某个数据库中的一个表

-F format的意思，压缩存储
-F c备份二进制格式
-F p 备份为文本，数据量大时不推荐

**pg_dumpall**

**copy**

```
[postgres@localhost ~]$ vi jan_test.txt
[postgres@localhost ~]$ cat jan_test.txt 
1    jan_1
2    jan_2
3    jan_3
[postgres@localhost ~]$ psql -U jan -d jan_db
psql (12.2)
Type "help" for help.

jan_db=> \d
       List of relations
 Schema | Name | Type  | Owner 
--------+------+-------+-------
 public | t1   | table | jan
(1 row)

jan_db=> create table jan_test as select * from t1 where 1=2;
SELECT 0
jan_db=> \d
         List of relations
 Schema |   Name   | Type  | Owner 
--------+----------+-------+-------
 public | jan_test | table | jan
 public | t1       | table | jan
(2 rows)

jan_db=> exit
[postgres@localhost ~]$ psql
psql (12.2)
Type "help" for help.


[postgres@localhost ~]$ psql -U jan -d jan_db
psql (12.2)
Type "help" for help.

jan_db=> \copy jan_test from /home/postgres/jan_test.txt
COPY 3
jan_db=> select * from jan_test;
 id | name  
----+-------
  1 | jan_1
  2 | jan_2
  3 | jan_3
(3 rows)
```

常见用法

```
\copy test_copy from /home/postgres/test_copy.txt;
\copy test_copy to /home/postgres/test_copy_1.txt;
生成以逗号tab为分隔符的平面文件
\copy test_copy to /home/postgres/test_copy_1.csv with csv;
生成以逗号“,”为分隔符的平面文件
\copy test_copy from /home/postgres/test_copy_1.csv with csv;
\copy test_copy from /home/postgres/test_copy.txt;
copy test_copy from '/home/postgres/test_copy.txt';
```

**pg_restore**

```
[postgres@localhost ~]$ ll |grep dm_bk
drwxrwxr-x. 2 postgres postgres        6 Sep 21 14:45 dm_bk

[postgres@localhost ~]$ psql -d jan_db -U jan
psql (12.2)
Type "help" for help.

jan_db=> create table t1(id int,name varchar(20));
CREATE TABLE


jan_db=> \d
       List of relations
 Schema | Name | Type  | Owner 
--------+------+-------+-------
 public | t1   | table | jan
(1 row)


jan_db=> insert into t1 values (generate_series(1, 100),'jan'||generate_series(1, 100));
INSERT 0 100


jan_db=> select * from t1 limit 3;
 id | name 
----+------
  1 | jan1
  2 | jan2
  3 | jan3
(3 rows)

jan_db=> exit


[postgres@localhost ~]$ pg_dump -F c -C jan_db -E UTF8 -h 127.0.0.1 -f dm_bk/jandb.dmp


[postgres@localhost ~]$ ll dm_bk/jandb.dmp
-rw-rw-r--. 1 postgres postgres 1571 Sep 21 15:09 dm_bk/jandb.dmp
[postgres@localhost ~]$ hexdump -e '16/1 "%02X " "  |  "' -e '16/1 "%_p" "\n"' dm_bk/jandb.dmp
50 47 44 4D 50 01 0E 00 04 08 01 01 01 00 00 00  |  PGDMP...........
00 1B 00 00 00 00 09 00 00 00 00 0F 00 00 00 00  |  ................
15 00 00 00 00 08 00 00 00 00 78 00 00 00 00 01  |  ..........x.....
00 00 00 00 06 00 00 00 6A 61 6E 5F 64 62 00 04  |  ........jan_db..
00 00 00 31 32 2E 32 00 04 00 00 00 31 32 2E 32  |  ...12.2.....12.2
00 06 00 00 00 00 11 0C 00 00 00 00 00 00 00 00  |  ................
01 00 00 00 30 00 01 00 00 00 30 00 08 00 00 00  |  ....0.....0.....
45 4E 43 4F 44 49 4E 47 00 08 00 00 00 45 4E 43  |  ENCODING.....ENC
4F 44 49 4E 47 00 02 00 00 00 00 1E 00 00 00 53  |  ODING..........S
45 54 20 63 6C 69 65 6E 74 5F 65 6E 63 6F 64 69  |  ET client_encodi
......
......


[postgres@localhost ~]$ pg_restore -l dm_bk/jandb.dmp 
;
; Archive created at 2020-09-21 15:09:27 EDT
;     dbname: jan_db
;     TOC Entries: 6
;     Compression: -1
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 12.2
;     Dumped by pg_dump version: 12.2
;
;
; Selected TOC Entries:
;
202; 1259 16419 TABLE public t1 jan
3086; 0 16419 TABLE DATA public t1 jan



[postgres@localhost ~]$ pg_restore -d jan_db dm_bk/jandb.dmp 


[postgres@localhost ~]$ psql -d jan_db -U jan
psql (12.2)
Type "help" for help.


jan_db=> \d
       List of relations
 Schema | Name | Type  | Owner 
--------+------+-------+-------
 public | t1   | table | jan
(1 row)


jan_db=> select count(*) from t1;
 count 
-------
   100
(1 row)


jan_db=> select * from t1 limit 3;
 id | name 
----+------
  1 | jan1
  2 | jan2
  3 | jan3
(3 rows)
[postgres@localhost ~]$ pg_restore -l -f jandb_more.toc dm_bk/jandb.dmp

[postgres@localhost ~]$ ll |grep jandb_more
-rw-rw-r--. 1 postgres postgres      388 Sep 21 15:17 jandb_more.toc
```

精细化导入

```
[postgres@localhost ~]$ vi jandb_more.toc 

;
; Archive created at 2020-09-21 15:09:27 EDT
;     dbname: jan_db
;     TOC Entries: 6
;     Compression: -1
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 12.2
;     Dumped by pg_dump version: 12.2
;
;
; Selected TOC Entries:
;
202; 1259 16419 TABLE public t1 jan
;3086; 0 16419 TABLE DATA public t1 jan



[postgres@localhost ~]$ psql
psql (12.2)
Type "help" for help.


postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 jan_db    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)


postgres=# create database jan_db_2;
CREATE DATABASE



postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 jan_db    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 jan_db_2  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(5 rows)

postgres=# exit


[postgres@localhost ~]$ pg_restore -F c -L jandb_more.toc -d jan_db_2 dm_bk/jandb.dmp 


[postgres@localhost ~]$ psql jan_db_2
psql (12.2)
Type "help" for help.


jan_db_2=# \d
       List of relations
 Schema | Name | Type  | Owner 
--------+------+-------+-------
 public | t1   | table | jan
(1 row)


jan_db_2=# select count(*) from t1;
 count 
-------
     0
(1 row)
```



### 本地外部表

创建平面文件

```
[postgres@localhost ~]$ psql -d jan_db -U jan
psql (12.2)
Type "help" for help.


jan_db=> \copy jan_test to /home/postgres/test_copy_1.csv with csv;
COPY 3
jan_db=> exit


[postgres@localhost ~]$ pwd
/home/postgres


[postgres@localhost ~]$ ll |grep test_copy_1
-rw-rw-r--. 1 postgres postgres       24 Sep 22 09:41 test_copy_1.csv


[postgres@localhost ~]$ cat test_copy_1.csv 
1,jan_1
2,jan_2
3,jan_3
```

安装外部表

```
jan_db=> create extension file_fdw;
ERROR:  could not open extension control file "/usr/local/pg12.2/share/postgresql/extension/file_fdw.control": No such file or directory


jan_db=> \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description          
---------+---------+------------+------------------------------
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)

jan_db=> exit



[postgres@localhost ~]$ cd /soft/postgresql-12.2/contrib/file_fdw/


[postgres@localhost file_fdw]$ make
make -C ../../src/backend generated-headers
make[1]: Entering directory `/soft/postgresql-12.2/src/backend'
make -C catalog distprep generated-header-symlinks
......
......


[postgres@localhost file_fdw]$ make install
make -C ../../src/backend generated-headers
make[1]: Entering directory `/soft/postgresql-12.2/src/backend'
make -C catalog distprep generated-header-symlinks
......
......
```

配置外部表

```
[postgres@localhost file_fdw]$ psql
psql (12.2)
Type "help" for help.


postgres=# \dx
                        List of installed extensions
    Name    | Version |   Schema   |              Description               
------------+---------+------------+----------------------------------------
 oracle_fdw | 1.1     | public     | foreign data wrapper for Oracle access
 plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)


postgres=# create extension file_fdw;
CREATE EXTENSION
postgres=# \dx
                         List of installed extensions
    Name    | Version |   Schema   |                Description                
------------+---------+------------+-------------------------------------------
 file_fdw   | 1.0     | public     | foreign-data wrapper for flat file access
 oracle_fdw | 1.1     | public     | foreign data wrapper for Oracle access
 plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
(3 rows)


postgres=# create server pg_file_server foreign data wrapper file_fdw;
CREATE SERVER


postgres=# create foreign table jan_test_file_fdw(id int,name varchar(20)) server pg_file_server
postgres-# options(filename '/home/postgres/test_copy_1.csv',format 'csv',header 'true',delimiter ',');
CREATE FOREIGN TABLE


postgres=# \d
                   List of relations
 Schema |       Name        |     Type      |  Owner   
--------+-------------------+---------------+----------
 public | jan_test_file_fdw | foreign table | postgres
 public | ora_emp           | foreign table | scott_pg
(2 rows)

postgres=# select * from jan_test_file_fdw;
 id | name  
----+-------
  2 | jan_2
  3 | jan_3
(2 rows)

postgres=# \d jan_test_file_fdw
                   Foreign table "public.jan_test_file_fdw"
 Column |         Type          | Collation | Nullable | Default | FDW options 
--------+-----------------------+-----------+----------+---------+-------------
 id     | integer               |           |          |         | 
 name   | character varying(20) |           |          |         | 
Server: pg_file_server
FDW options: (filename '/home/postgres/test_copy_1.csv', format 'csv', header 'true', delimiter ',')

postgres=# 
```



### 热备

理论

1. 强制日志full-page write模式
2. 切换当前WAL端文件（version 8.4 or later）
3. 发生检查点，将dirty buffer落盘
4. 创建一个位于base目录同层的、包含备份基本信息的标记文件
   ![image.png](http://cdn.lifemini.cn/dbblog/20200922/c81f4e78d67e47a79c564b7db2eee341.png)

备份前准备

```
[postgres@localhost file_fdw]$ cd $PG_HOME/data 
[postgres@localhost data]$ vi postgresql.conf

archive_mode = on
archive_command = 'cp %p /home/postgres/arch/%f'
archive_cleanup_command = 'pg_archivecleanup /home/postgres/arch %r'
```

<span style=color:red>**注意**</span>

<span style=color:red>1. 多个数据库的备份不要放在一起，否则恢复时容易发生错误。</span>

开始热备

```
[postgres@localhost data]$ psql
psql (12.2)
Type "help" for help.

postgres=# select pg_start_backup('baseline');
 pg_start_backup 
-----------------
 0/E000028
(1 row)








[postgres@localhost ~]$ cd $PG_HOME/data
[postgres@localhost data]$ cat backup_label
START WAL LOCATION: 0/C000028 (file 00000001000000000000000C)
CHECKPOINT LOCATION: 0/C000060
BACKUP METHOD: pg_start_backup
BACKUP FROM: master
START TIME: 2020-09-22 10:06:22 EDT
LABEL: baseline
START TIMELINE: 1






[postgres@localhost data]$ tar -jcv -f /home/postgres/online_bk/baseline.tar.bz2 $PGDATA
......
......
/usr/local/pg12.2/data/postmaster.opts
/usr/local/pg12.2/data/backup_label.old
/usr/local/pg12.2/data/postgresql.conf
/usr/local/pg12.2/data/postmaster.pid
/usr/local/pg12.2/data/current_logfiles
/usr/local/pg12.2/data/backup_label

[postgres@localhost online_bk]$ pwd
/home/postgres/online_bk
[postgres@localhost online_bk]$ ll
total 4256
-rw-rw-r--. 1 postgres postgres 4357164 Sep 22 10:09 baseline.tar.bz2


[postgres@localhost data]$ psql
psql (12.2)
Type "help" for help.

postgres=# select pg_stop_backup();
NOTICE:  all required WAL segments have been archived
 pg_stop_backup 
----------------
 0/C000138
(1 row)

postgres=# exit
[postgres@localhost data]$ pwd
/usr/local/pg12.2/data
[postgres@localhost data]$ ll |grep backup_label
-rw-rw-r--. 1 postgres postgres   231 Sep 16 12:55 backup_label.old
```

### 物理备份