---
title:[PG]--后台进程与系统表
date:2020-09-18
---



##### [PG]--后台进程与系统表

- [进程结构](进程结构)
- [系统表](系统表)



### 进程结构

![image.png](http://cdn.lifemini.cn/dbblog/20200917/fce94f6f09864047a1d0f34467d0ef97.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200917/9cd32bceaa26455bbc3dd39dfeb6faf7.png)



### 系统表

系统表与系统表之间是以Oid相连的。

```
postgres=# select relkind,relname from pg_class where relnamespace = (select oid from pg_namespace where nspname='pg_catalog') and relkind='r' order by 1,2;
 relkind |         relname         
---------+-------------------------
 r       | pg_aggregate
 r       | pg_am
 r       | pg_amop
 r       | pg_amproc
 r       | pg_attrdef
 r       | pg_attribute
 r       | pg_auth_members
......
......
```

pg_aggrate 聚合函数信息
pg_am 系统支持的索引信息

```
postgres=# select amname from pg_am;
 amname 
--------
 heap
 btree
 hash
 gist
 gin
 spgist
 brin
(7 rows)
```

pg_amop存储每个索引的操作符家族
pg_amproc存储每个索引访问方法操作符家族
pg_attribute才能出数据列的详细信息(ctid,cmin,cmax,xmin,xmax）
pg_auth_members数据库用户成员信息
pg_authid存储数据库用户详细信息
pg_cast数据库显性类型转换路径信息
pg_class数据库所有对象信息
pg_collation集信息，encoding,collate,ctype等
pg_constraint存储定义的约束信息
pg_conversion字符集之间的转换信息
pg_database集群中数据库的信息
pg_db_role_setting基于角色和数据库组合的定制信息
pg_default_acl存储新建对象的初始化权限信息
pg_depend数据库对象之间的依赖信息
pg_description数据库对象之间的描述信息
pg_enum枚举类型信息
pg_event_trigger时间触发器信息
pg_extension扩展插件信息
pg_foreign_data_wrapper FDW信息
pg_foreign_table外部表信息
pg_foreign_server外部表服务器信息
pg_index索引信息
pg_inherits继承表的集成关系信息
pg_language过程语言信息
pg_largeobject大对象切片后的真实数据存储在这个表里
pg_largeobject_metadata大对象辕信息，如owner,访问权限
pg_namespace数据库中的schema信息
pg_opclass索引访问方法的操作符分类信息
pg_operator操作符信息
pg_opfamily操作符家族信息
pg_pltemplate过程语言的模板信息
pg_proc数据库服务端函数信息
pg_range范围类型信息
pg_rewrite表和视图的重写规则信息
pg_seclabel安全标签信息（SELinux）
pg_shdepend数据库中的对象之间或集群共享对象之间的依赖关系
pg_description共享对象的描述信息
pg_shseclabel共享对象的安全标签信息（SELinux）
pg_statistic analyze生成的统计信息，用于查询计划计算成本
pg_tablespace表空间相关信息
pg_trigger表上的触发器信息
pg_ts_config全文检索配置信息
pg_ts_config全文检索配置映射信息
pg_ts_dict全文检索字典信息
pg_ts_parser全文检索分析器信息
pg_ts_remplate全文检索模板信息
pg_type数据库中的类型信息
pg_user_mapping foreign server的用户配置信息
v | pg_available_extension_versions
v | pg_available_extensions
v | pg_config
v | pg_cursors
v | pg_file_settings
v | pg_group
v | pg_hba_file_rules
v | pg_indexes
v | pg_locks
v | pg_matviews
v | pg_policies
v | pg_prepared_statements
v | pg_prepared_xacts
v | pg_publication_tables
v | pg_replication_origin_status
v | pg_replication_slots
v | pg_roles
v | pg_rules
v | pg_seclabels
v | pg_sequences
v | pg_settings
v | pg_shadow
v | pg_stat_activity
v | pg_stat_all_indexes
v | pg_stat_all_tables
v | pg_stat_archiver
v | pg_stat_bgwriter
v | pg_stat_database
v | pg_stat_database_conflicts
v | pg_stat_gssapi
v | pg_stat_progress_cluster
v | pg_stat_progress_create_index
v | pg_stat_progress_vacuum
v | pg_stat_replication
v | pg_stat_ssl
v | pg_stat_subscription
v | pg_stat_sys_indexes
v | pg_stat_sys_tables
v | pg_stat_user_functions
v | pg_stat_user_indexes
v | pg_stat_user_tables
v | pg_stat_wal_receiver
v | pg_stat_xact_all_tables
v | pg_stat_xact_sys_tables
v | pg_stat_xact_user_functions
v | pg_stat_xact_user_tables
v | pg_statio_all_indexes
v | pg_statio_all_sequences
v | pg_statio_all_tables
v | pg_statio_sys_indexes
v | pg_statio_sys_sequences
v | pg_statio_sys_tables
v | pg_statio_user_indexes
v | pg_statio_user_sequences
v | pg_statio_user_tables
v | pg_stats
v | pg_stats_ext
v | pg_tables
v | pg_timezone_abbrevs
v | pg_timezone_names
v | pg_user
v | pg_user_mappings
v | pg_views

隐式转换

```
postgres=# select * from pg_attribute where attrelid='ora_emp'::regclass;
 attrelid | attname  | atttypid | attstattarget | attlen | attnum | attndims | attcacheoff | 
atttypmod | attbyval | attstorage | attalign | attnotnull | atthasdef | atthasmissing | attid
entity | attgenerated | attisdropped | attislocal | attinhcount | attcollation | attacl | att
options | attfdwoptions | attmissingval 
----------+----------+----------+---------------+--------+--------+----------+-------------+-
----------+----------+------------+----------+------------+-----------+---------------+------
-------+--------------+--------------+------------+-------------+--------------+--------+----
--------+---------------+---------------
    16403 | tableoid |       26 |             0 |      4 |     -6 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | cmax     |       29 |             0 |      4 |     -5 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | xmax     |       28 |             0 |      4 |     -4 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | cmin     |       29 |             0 |      4 |     -3 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | xmin     |       28 |             0 |      4 |     -2 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | ctid     |       27 |             0 |      6 |     -1 |        0 |          -1 | 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# select oid,relname from pg_class where relname='ora_emp';
  oid  | relname 
-------+---------
 16403 | ora_emp
(1 row)

postgres=# select * from pg_attribute where attrelid=16403;
 attrelid | attname  | atttypid | attstattarget | attlen | attnum | attndims | attcacheoff | 
atttypmod | attbyval | attstorage | attalign | attnotnull | atthasdef | atthasmissing | attid
entity | attgenerated | attisdropped | attislocal | attinhcount | attcollation | attacl | att
options | attfdwoptions | attmissingval 
----------+----------+----------+---------------+--------+--------+----------+-------------+-
----------+----------+------------+----------+------------+-----------+---------------+------
-------+--------------+--------------+------------+-------------+--------------+--------+----
--------+---------------+---------------
    16403 | tableoid |       26 |             0 |      4 |     -6 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | cmax     |       29 |             0 |      4 |     -5 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | xmax     |       28 |             0 |      4 |     -4 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | cmin     |       29 |             0 |      4 |     -3 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | xmin     |       28 |             0 |      4 |     -2 |        0 |          -1 | 
       -1 | t        | p          | i        | t          | f         | f             |      
       |              | f            | t          |           0 |            0 |        |    
        |               | 
    16403 | ctid     |       27 |             0 |      6 |     -1 |        0 |          -1 | 
postgres=# 
```

relkind
r-->ordinary table
i-->index
S-->sequence
v-->view
m-->materialized view
c-->composite type
t-->TOAST
f-->foreign table

```
postgres=# select * from pg_db_role_setting;
 setdatabase | setrole | setconfig 
-------------+---------+-----------
(0 rows)

postgres=# create user jan with password 'jan';
CREATE ROLE

postgres=# alter role jan set enable_seqscan=off;
ALTER ROLE

postgres=# select * from pg_db_role_setting;
 setdatabase | setrole |      setconfig       
-------------+---------+----------------------
           0 |   16409 | {enable_seqscan=off}
(1 row)
postgres=# select * from pg_default_acl;
 oid | defaclrole | defaclnamespace | defaclobjtype | defaclacl 
-----+------------+-----------------+---------------+-----------
(0 rows)

postgres=# \dn
  List of schemas
  Name  |  Owner   
--------+----------
 public | postgres
(1 row)


postgres=# create schema jan;
CREATE SCHEMA


postgres=# \dn
  List of schemas
  Name  |  Owner   
--------+----------
 jan    | postgres
 public | postgres
(2 rows)

postgres=# grant select on all tables in schema jan to jan;
GRANT

postgres=# select * from pg_default_acl;
 oid | defaclrole | defaclnamespace | defaclobjtype | defaclacl 
-----+------------+-----------------+---------------+-----------
(0 rows)


postgres=# alter default privileges for role jan in schema jan grant select on tables to jan; 
ALTER DEFAULT PRIVILEGES

postgres=# select * from pg_default_acl;
  oid  | defaclrole | defaclnamespace | defaclobjtype |  defaclacl  
-------+------------+-----------------+---------------+-------------
 16414 |      16409 |           16410 | r             | {jan=r/jan}
```

仅与alter default privileges for role jan in schema jan grant select on tables to jan;这句话有关

```
postgres=# create function f_jan(id int) returns int as $$
postgres$# declare
postgres$# begin
postgres$# return id+i;
postgres$# end;
postgres$# $$ language plpgsql strict;
CREATE FUNCTION


postgres=# select * from pg_proc where proname='f_jan';
  oid  | proname | pronamespace | proowner | prolang | procost | prorows | provariadic | pros
upport | prokind | prosecdef | proleakproof | proisstrict | proretset | provolatile | propara
llel | pronargs | pronargdefaults | prorettype | proargtypes | proallargtypes | proargmodes |
 proargnames | proargdefaults | protrftypes |    prosrc    | probin | proconfig | proacl 
-------+---------+--------------+----------+---------+---------+---------+-------------+-----
-------+---------+-----------+--------------+-------------+-----------+-------------+--------
-----+----------+-----------------+------------+-------------+----------------+-------------+
-------------+----------------+-------------+--------------+--------+-----------+--------
 16415 | f_jan   |         2200 |       10 |   13583 |     100 |       0 |           0 | -   
       | f       | f         | f            | t           | f         | v           | u      
     |        1 |               0 |         23 | 23          |                |             |
 {id}        |                |             |             +|        |           | 
       |         |              |          |         |         |         |             |     
       |         |           |              |             |           |             |        
     |          |                 |            |             |                |             |
             |                |             | declare     +|        |           | 
       |         |              |          |         |         |         |             |     
       |         |           |              |             |           |             |        
     |          |                 |            |             |                |             |
             |                |             | begin       +|        |           | 
       |         |              |          |         |         |         |             |     
       |         |           |              |             |           |             |        
     |          |                 |            |             |                |             |
             |                |             | return id+i;+|        |           | 
       |         |              |          |         |         |         |             |     
       |         |           |              |             |           |             |        
     |          |                 |            |             |                |             |
             |                |             | end;        +|        |           | 
       |         |              |          |         |         |         |             |     


postgres=# \x
Expanded display is on.
postgres=# select * from pg_proc where proname='f_jan';
-[ RECORD 1 ]---+-------------
oid             | 16415
proname         | f_jan
pronamespace    | 2200
proowner        | 10
prolang         | 13583
procost         | 100
prorows         | 0
provariadic     | 0
prosupport      | -
prokind         | f
prosecdef       | f
proleakproof    | f
proisstrict     | t
proretset       | f
provolatile     | v
proparallel     | u
pronargs        | 1
pronargdefaults | 0
prorettype      | 23
proargtypes     | 23
proallargtypes  | 
proargmodes     | 
proargnames     | {id}
proargdefaults  | 
protrftypes     | 
prosrc          |             +
                | declare     +
                | begin       +
postgres=# select relkind,relname from pg_class where relnamespace=(select oid from pg_namespace where nspname='pg_catalog') and relkind='v' order by 1,2;
-[ RECORD 63 ]---------------------------
relkind | v
relname | pg_views

 \dvS
-[ RECORD 63 ]--------------------------
Schema | pg_catalog
Name   | pg_views
Type   | view
Owner  | postgres
```