---
title:[PG]--Postgresql_FDW下载、安装与使用
date:2020-08-29
---



### 下载FDW

**下载地址**：

> 1. [Github下载地址](https://github.com/laurenz/oracle_fdw/releases)
> 2. [JanNest下载地址](https://pan.baidu.com/s/1jjtTtGr1PmR0-Q0XgSikcw) 提取码：old2

<span style=color:red>**注意：**</span>

<span style=color:red>1. 安装对应自己版本的FDW</span>
<span style=color:red>2. 其他FDW源代码兼容PostgreSQL的FDW</span>
   例如：想要构建PG到PG之间的DB link访问，那么可以下载任意FDW源码安装。



### 安装FDW

将源码压缩包上传至/soft目录后解压，将解压后文件移动到安装pg时的源码目录。

```
[root@localhost soft]# unzip oracle_fdw-ORACLE_FDW_2_2_0.tar.gz 

[root@localhost contrib]# mv oracle_fdw-ORACLE_FDW_2_2_0 /soft/postgresql-12.2/contrib/

[root@localhost contrib]# cd /soft/postgresql-12.2/contrib/
```

在contrib目录下正式开始编译、安装。

```
[root@localhost contrib]# make

[root@localhost contrib]# make install
```



### 配置FDW

使用端IP：192.168.1.30
目标端IP：192.168.1.50

**使用端：**
使用psql查看FDW是否安装成功。

```
postgres=# select * from pg_available_extensions;
...
postgres_fdw | 1.0 | 1.0 | foreign-data wrapper for remote PostgreSQL servers
...



postgres=# \dx
                               List of installed extensions
     Name     | Version |   Schema   |                    Description                     
--------------+---------+------------+----------------------------------------------------
 plpgsql      | 1.0     | pg_catalog | PL/pgSQL procedural language
 postgres_fdw | 1.0     | public     | foreign-data wrapper for remote PostgreSQL servers
(2 rows)
```

**目标端：**
在目标端创建测试使用的数据库，并创建测试数据。

```
[postgres@pg1 data]$ psql
psql (12.2)
Type "help" for help.

postgres=# create user januser with password '123123';
CREATE ROLE

postgres=> select rolname,rolpassword,oid from pg_roles;
          rolname          | rolpassword |  oid  
---------------------------+-------------+-------
...
 januser                   | ********    | 49161
...


postgres=# CREATE DATABASE jantestdb;
CREATE DATABASE


postgres=# alter database jantestdb owner to januser;
ALTER DATABASE



postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
...
 jantestdb | januser  | UTF8     | en_US.utf8 | en_US.utf8 | 
...
```

**目标端：**
在目标端创建测试表、及测试数据

```
[postgres@pg2 data]$ psql -Ujanuser jantestdb
psql (12.2)
Type "help" for help.

jantestdb=> create table test(id int,name varchar(20));
CREATE TABLE

jantestdb=> insert into test values (1,'Jan01');
INSERT 0 1

jantestdb=> select * from test;
 id | name  
----+-------
  1 | Jan01
(1 row)
```

**使用端：**
在使用端创建FDW服务、配置目标端的用户信息、创建外部表。

```
postgres=# create server server_remote_50 foreign data wrapper postgres_fdw options(host '192.168.1.50',port '5432',dbname 'test');
CREATE SERVER


postgres=# create user mapping for postgres server server_remote_50 options(user 'januser',password '123123');
CREATE USER MAPPING


postgres=# CREATE FOREIGN TABLE test_fdw(id int,name varchar(20)) server server_remote_50 options (schema_name 'public',table_name 'test');
CREATE FOREIGN TABLE
```



### 验证FDW

**使用端**

```
postgres=# select * from test_fdw;
 id |  name  
----+--------
  1 | Jan01
(1 rows)


postgres=# insert into test_fdw values (2,'Jan02');
INSERT 0 1


postgres=# select * from test_fdw;
 id |  name  
----+--------
  1 | Jan01
  2 | Jan02
(2 rows)
```

**目标端：**

```
jantestdb=> select * from test;
 id | name  
----+-------
  1 | Jan01
  2 | Jan02
(2 row)
```



### 其他操作

```
-- 查询已创建的连接
SELECT * from pg_user_mappings;


-- 删除创建的对象
drop foreign table test_fdw;
drop user mapping for highgo server  postgres;
drop server server_remote_50;
```