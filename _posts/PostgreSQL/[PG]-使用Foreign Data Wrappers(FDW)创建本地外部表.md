---
title:[PG]-使用Foreign Data Wrappers(FDW)创建本地外部表
date:2020-11-9
---



​		PG的copy命令产生一个文本文件，然后创建一个本地的Foreign Data Wrappers(FDW)

### 生成外部文件

```
postgres=# Create table szp_copy_test(id int,name varchar(20));
CREATE TABLE
postgres=# Insert into szp_copy_test values(1,'fdw_1');
INSERT 0 1
postgres=# Insert into szp_copy_test values(2,'fdw_2');
INSERT 0 1
postgres=# \copy szp_copy_test to /home/postgres/szp_copy_test.txt;
COPY 2
postgres=# exit

[postgres@pg1 ~]$ cat szp_copy_test.txt
1    fdw_1
2    fdw_2
```



### 创建FDW拓展

```
[postgres@pg1 ~]$ psql
psql (12.2)
Type "help" for help.

postgres=# Create extension file_fdw;
CREATE EXTENSION
postgres=# \dx
                                List of installed extensions
    Name     | Version |   Schema   |                      Description                      
-------------+---------+------------+-------------------------------------------------------
 file_fdw    | 1.0     | public     | foreign-data wrapper for flat file access
 pageinspect | 1.7     | public     | inspect the contents of database pages at a low level
 plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
(3 rows)
postgres=# Create server szp_file_server foreign data wrapper file_fdw;
CREATE SERVER
postgres=# Create foreign table szp_copy_test_fdw(id int,name varchar(20)) server szp_file_server options(filename '/home/postgres/szp_copy_test.txt',header 'false');
CREATE FOREIGN TABLE

postgres=# \d szp_copy_test_fdw
                   Foreign table "public.szp_copy_test_fdw"
 Column |         Type          | Collation | Nullable | Default | FDW options 
--------+-----------------------+-----------+----------+---------+-------------
 id     | integer               |           |          |         | 
 name   | character varying(20) |           |          |         | 
Server: szp_file_server
FDW options: (filename '/home/postgres/szp_copy_test.txt', header 'false')
```



### 验证创建

```
postgres=# Select * from szp_copy_test_fdw;
 id | name  
----+-------
  1 | fdw_1
  2 | fdw_2
(2 rows)
```