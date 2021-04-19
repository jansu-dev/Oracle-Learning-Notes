---
title:[TiDB]-使用管理之用户管理
date:2020-12-20
---



[TiDB]-使用管理之用户管理



### 用户管理

增
删

授权用户

服务及修改root密码



### 权限系统表

| 表名         | 说明                         |
| ------------ | ---------------------------- |
| user         | 用户账户、全局权限及非权限列 |
| db           | 数据库级别权限               |
| table_priv   | 表级别权限                   |
| columns_priv | 列级别权限                   |



### 统计信息系统表

| 表名             | 说明                         |
| ---------------- | ---------------------------- |
| stats_buckets    | 统计信息的桶                 |
| stats_histograms | 统计信息直方图               |
| stats_meta       | 表的元信息（总行数、修改数） |



### GC Worker相关表

| 表名            | 说明              |
| --------------- | ----------------- |
| gc_delete_range | 查看gc worker状态 |



## 其他系统表

| 表名             | 说明                         |
| ---------------- | ---------------------------- |
| GLOBAL_VARIABLES | 全局系统表（主要以tidb开头） |



### 具体操作

#### 创建、授权和删除用户

使用 CREATE USER 语句创建一个用户 tiuser，密码为 123456：

```
CREATE USER 'tiuser'@'localhost' IDENTIFIED BY '123456';
```

授权用户 tiuser 可检索数据库 samp_db 内的表：

```
GRANT SELECT ON samp_db.* TO 'tiuser'@'localhost';
```

查询用户 tiuser 的权限：

```
SHOW GRANTS for tiuser@localhost;
```

删除用户 tiuser：

```
DROP USER 'tiuser'@'localhost';
```



### 常见用户管理命令

```
grant all on *.* to jan@'%';

crate user 'jan'@'%' identified by '123123';

grant select,insert,update,delete on *.* to jan@'%';
```



### 修改密码

```
set password for jan@'%' = password('123123');
```



### 权限系统表

```
select * from mysql.user;
```



### 全局变量系统表

```
select * from GLOBAL_VARIABLES;

select * from GLOBAL_VARIABLES where  lik2 '%%';
```



### 查看进程

```
MySQL [(none)]> show processlist;
+------+------+--------------+------+---------+------+-------+------------------+
| Id   | User | Host         | db   | Command | Time | State | Info             |
+------+------+--------------+------+---------+------+-------+------------------+
|   27 | root | 192.168.1.41 | NULL | Query   |    0 | 2     | show processlist |
+------+------+--------------+------+---------+------+-------+------------------+
1 row in set (0.00 sec)
```



### 查看ddl

```
MySQL [(none)]> admin show ddl jobs;
+--------+---------+-------------------------+---------------+--------------+-----------+----------+-----------+-----------------------------------+--------+
| JOB_ID | DB_NAME | TABLE_NAME              | JOB_TYPE      | SCHEMA_STATE | SCHEMA_ID | TABLE_ID | ROW_COUNT | START_TIME                        | STATE  |
+--------+---------+-------------------------+---------------+--------------+-----------+----------+-----------+-----------------------------------+--------+
|     46 | jan     | jan_test                | drop index    | none         |        41 |       43 |         0 | 2020-12-27 01:48:32.744 +0800 CST | synced |
|     45 | jan     | jan_test                | add index     | public       |        41 |       43 |         1 | 2020-12-27 01:48:13.395 +0800 CST | synced |
|     44 | jan     | jan_test                | create table  | public       |        41 |       43 |         0 | 2020-12-23 23:09:44.207 +0800 CST | synced |
|     42 | jan     |                         | create schema | public       |        41 |        0 |         0 | 2020-12-23 22:08:47.8 +0800 CST   | synced |
|     40 | mysql   | expr_pushdown_blacklist | create table  | public       |         3 |       39 |         0 | 2020-12-23 22:07:14.7 +0800 CST   | synced |
|     38 | mysql   | stats_top_n             | create table  | public       |         3 |       37 |         0 | 2020-12-23 22:07:14.4 +0800 CST   | synced |
|     36 | mysql   | bind_info               | create table  | public       |         3 |       35 |         0 | 2020-12-23 22:07:14.3 +0800 CST   | synced |
|     34 | mysql   | default_roles           | create table  | public       |         3 |       33 |         0 | 2020-12-23 22:07:14.199 +0800 CST | synced |
|     32 | mysql   | role_edges              | create table  | public       |         3 |       31 |         0 | 2020-12-23 22:07:14.1 +0800 CST   | synced |
|     30 | mysql   | stats_feedback          | create table  | public       |         3 |       29 |         0 | 2020-12-23 22:07:12.7 +0800 CST   | synced |
+--------+---------+-------------------------+---------------+--------------+-----------+----------+-----------+-----------------------------------+--------+
10 rows in set (0.00 sec)

MySQL [(none)]> 
```



### 查看慢查询

```
MySQL [(none)]> select sleep(3);
+----------+
| sleep(3) |
+----------+
|        0 |
+----------+
1 row in set (3.00 sec)

MySQL [(none)]> admin show slow top 3\G
*************************** 1. row ***************************
           SQL: select sleep(3)
         START: 2020-12-27 16:55:53.483093
      DURATION: 00:00:03
       DETAILS: 
          SUCC: 1
       CONN_ID: 27
TRANSACTION_TS: 0
          USER: root@192.168.1.41
            DB: 
     TABLE_IDS: 
     INDEX_IDS: 
      INTERNAL: 0
        DIGEST: b4dae6a771c1d84157dcc302bef38cbff77a7a8ff89ee38302ac3324485454a3
1 row in set (0.00 sec)
```



### 分区表特性

range分区

```
create table jan_range_datetime(id int,hiredate datetime)
partition by range(to_days(hiredate))(
  partition p1 values less than (to_days('20201202')),
  partition p2 values less than (to_days('20201203')),
  partition p3 values less than (to_days('20201204')),
  partition p4 values less than (to_days('20201205')),
  partition p5 values less than (to_days('20201206')),
  partition p6 values less than (to_days('20201207')),
  partition p7 values less than (to_days('20201208')),
  partition p8 values less than (to_days('20201209')),
  partition p9 values less than (to_days('20201210')),
  partition p10 values less than (to_days('20201211')),
  partition p11 values less than (maxvalue)
);
```

hash分区

```
create table jan_hash_member(
  id int not null,
  fname varchar(20),
  lname varchar(20),
  created date not null default '1970-01-01',
  separted date not null default '1998-03-06',
  job_code int,
  store_id int
)
partition by hash(id)
partitions 4;
```