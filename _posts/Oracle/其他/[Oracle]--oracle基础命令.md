---
title:[Oracle]--oracle基础命令
date:2020-07-02
---



#### 目录

- ##### [一、 基本命令](一、 基本命令)

- ##### [二、 参数文件相关](二、 参数文件相关)

- ##### [三、数据库启动与关闭命令](三、数据库启动与关闭命令)

- ##### [四、11g新特性ADR相关](四、11g新特性ADR相关)

- ##### [五、控制文件相关](五、控制文件相关)

- ##### [六、存储架构相关](六、存储架构相关)

- ##### [七、修改数据库配置相关](七、修改数据库配置相关)

- ##### [八、临时表空间](八、临时表空间)

- ##### [九、分区表相关](九、分区表相关)





#### oracle基础命令

### 一、 基本命令

1. ##### 非标准块存储表

```
alter table scott.emp1 storage(buffer_pool keep);
select segment_name,buffer_pool from dba_segments where segment_name='EMP1';

alter system set db_16k_cache_size=8m;
create tablespace tbs_16k datafile '/u01/oradata/prod/tbs16k01.dbf' size 10m blocksize 16k;
select TABLESPACE_NAME,block_size from dba_tablespaces;
```

1. ##### Oracle的进程

```
ps -ef |grep sqlplus
ps -ef |grep LOCAL
select pid,program,background from v$process;
```



### 二、 参数文件相关

1. ##### 生成参数文件

```
create pfile from spfile
create spfile from pfile
create pfile from memory;
create spfile from memory;
```

2. ##### 查看参数文件信息

```
show parameter spfile
NAME                      TYPE        VALUE
---------------------- ----------- ------------------------------
spfile                   string      /u01/oracle/dbs/spfile.ora

select name,value,isspecified from v$spparameter where name like 'memory_target';
NAME            VALUE             ISSPECIFIED
-----------   --------------- ---------------------
memory_target  423624704            TRUE
```



### 三、数据库启动与关闭命令

```
startup force;                            =shutdown abort + startup
startup upgrade                           只有sysdba能连接
startup restrict                      有restrict session权限才可登录，sys不受限制
alter system enable restricted session;   open后再限制
alter database open read only;            scn不会增长

shutdown normal                          拒绝新的连接，等待当前会话结束，ckpt
shutdown transactional                   拒绝新的连接，等待当前事务结束，ckpt
shutdown immediate                       拒绝新的连接，未提交的事务回滚，ckpt
shutdown abort                           实例恢复
```



四、11g新特性ADR相关

```
SQL> show parameter diagnostic_dest
NAME                              TYPE        VALUE
------------------------------------ ----------- ------------------------------
diagnostic_dest                      string      /u01

SQL> show parameter dump
SQL> select * from v$diag_info;
$cat /dev/null > alert_prod.log 直接删掉下次启动会自动创建
```



### 五、控制文件相关

1. ##### 
```
show parameter control_file 
select name from v$controlfile; 
v$controlfile_record_section
```

2. ##### 在跟踪文件中查看控制文件内容

```
alter session set events 'immediate trace name controlf level 12';
select * from v$diag_info;
```

3. ##### 备份并重建控制文件

```
alter database backup controlfile to '/u01/oradata/prod/con.bak';
alter database backup controlfile to trace;
alter database backup controlfile to trace as '/u01/oradata/prod/con.trace';

alter database backup controlfile to trace as '/u01/oradata/prod/con.trace';
startup force nomount
SQL>CREATE CONTROLFILE REUSE DATABASE "prod" NORESETLOGS  ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u01/oradata/prod/redo01.log'  SIZE 50M,
  GROUP 2 '/u01/oradata/prod/redo02.log'  SIZE 50M,
  GROUP 3 '/u01/oradata/prod/redo03.log'  SIZE 50M
-- STANDBY LOGFILE
DATAFILE
  '/u01/oradata/prod/system01.dbf',
  '/u01/oradata/prod/sysaux01.dbf',
  '/u01/oradata/prod/users01.dbf',
  '/u01/oradata/prod/example01.dbf',
  '/u01/oradata/prod/test01.dbf',
  '/u01/oradata/prod/undotbs01.dbf'
CHARACTER SET ZHS16GBK
;
```



### 六、存储架构相关

1. ##### 查询块头信息（数据文件、控制文件）

```
select file#,checkpoint_change# from v$datafile;
select file#,checkpoint_change# from v$datafile_header;
```

2. ##### 创建临时表空间

```
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/oradata/prod/temp01.dbf' SIZE 30408704  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
```

3. ##### 创建、切换、删除undo表空间

```
create undo tablespace undotbs2 datafile '/u01/oradata/prod/undotbs02.dbf' size 50m autoextend on;
alter system set undo_tablespace=undotbs2;
drop tablespace undotbs1 including contents and datafiles;
```

4. ##### 创建bigfile表空间、加数据文件

```
create bigfile tablespace big_tbs datafile '/u01/oradata/prod/bigtbs01.dbf' size 100m;
alter tablespace big_tbs add datafile '/u01/oradata/prod/bigtbs02.dbf' size 100m;
```

5. ##### 预分配段空间

```
alter table scott.t1 allocate extent (datafile '/u01/oradata/prod/test01.dbf' size 5m);
```

6. ##### deallocate回收free extent

```
alter table scott.t1 deallocate unused; 收回从未使用的extent
```

7. ##### 创建延迟段

```
create table t2(id int,name char(10)) SEGMENT CREATION deferred;
```

8. ##### 建表立即占用段空间

```
create table scott.t1(id int,name char(10)) SEGMENT CREATION IMMEDIATE TABLESPACE TB1;
```



### 七、修改数据库配置相关

- ##### 添加日志组

```
alter database add logfile group 4 '/u01/oradata/prod/redo04.log' size 50m;
```

- ##### 添加日志成员

```
mkdir -p /u01/disk2/prod
alter database add logfile member
'/u01/disk2/prod/redo01b.log' to group 1,
'/u01/disk2/prod/redo02b.log' to group 2,
'/u01/disk2/prod/redo03b.log' to group 3,
'/u01/disk2/prod/redo04b.log' to group 4;
```

- ##### 重建日志组

重建日志组

```
alter database clear logfile group 4;
```

非归档重建日志组

```
alter database clear unarchived logfile group n;
```

  1. 在v$log中可以查看到archived的状态(YES/NO)，YES说明ARCn进程已经将redo log file归档完毕，NO与之相反。
  2. 此处的状态将影响数据库实例恢复的过程，因Oracle是先从归档文件中开始恢复直至无可用文件为止，这样就可能回造成redo log file的current日志组中残留部分可恢复的信息。
  3. 如果此时强行resetlogs开库，会造成数据非最新数据，既发生了数据库的不完全恢复。

- ##### 删除当前日志组既日志组成员

```
alter database drop logfile group 4;   状态是current和active的组不能删除
alter database drop logfile member '/u01/disk2/prod/redo01b.log';
```

- ##### 设置归档路径与归档格式

```
alter system set log_archive_dest_1='location=/u01/arch';
alter system set log_archive_dest_2='service=standby';

alter system set log_archive_format ='arch_%t_%r_%s.log' scope=spfile;
%t   thread# 日志线程号
%s   sequence 日志序列号
%r   resetlog 代表数据库的周期
```

1. log_archive_dest、log_archive_duplex_dest，Oracle11g两个参数已弃用了，仅能完成两路复用但无法指定远程。
2. 使用log_archive_dest_n后log_archive_dest参数将失效了。

- ##### 切换归档

```
alter system switch logfile;   仅切换当前实例，适用归档和非归档。
alter system archive log current; 在RAC下切换所有实例，仅适用于归档模式。
```

- ##### 判定行迁移

```
analyze table t1 compute statistics;
select pct_free,pct_used,avg_row_len,chain_cnt,blocks from user_tables where table_name='T1';
PCT_FREE   PCT_USED AVG_ROW_LEN  CHAIN_CNT     BLOCKS
---------- ---------- ----------- ---------- ----------
10                      3          0              5
```

CHAIN_CNT有值时，查看AVG_ROW_LEN，
AVG_ROW_LEN<块大小：
\- 发生的是行迁移（行迁移块数量CHAIN_CNT）；
\- 否则可能有行链接。

- ##### 判定行连接

```
select distinct file#,block# from v$bh a,user_objects b where a.objd=b.object_id and b.object_name='T1' order by 2;
analyze table t1 compute statistics;
select pct_free,pct_used,avg_row_len,chain_cnt,blocks from user_tables where table_name='T1';


@/u01/oracle/rdbms/admin/utlchain.sql

analyze table scott.t1 LIST CHAINED ROWS;

select count(*) from chained_rows;

  COUNT(*)
----------
       865
select table_name, HEAD_ROWID from chained_rows where rownum<=3;
TABLE_NAME                     HEAD_ROWID
------------------------------ ------------------
T1                             AAASC4AAEAAAAIfABQ
T1                             AAASC4AAEAAAAIfABR
T1                             AAASC4AAEAAAAIfABS
Select dbms_rowid.ROWID_RELATIVE_FNO(rowid) fn,
dbms_rowid.rowid_block_number(rowid) bn, rowid,c1 from scott.t1 where rowid='AAASPhAAEAAAAIdABQ';
        FN         BN ROWID              C1
---------- ---------- ------------------ --------------------
         4        541 AAASPhAAEAAAAIdABQ prod is my name
drop table chained_rows;
```

- ##### 解决行迁移

```
1
alter table t1 move;
analyze table t1 compute statistics;
select pct_free,pct_used,avg_row_len,chain_cnt,blocks from user_tables where table_name='T1';
PCT_FREE   PCT_USED AVG_ROW_LEN  CHAIN_CNT     BLOCKS
---------- ---------- ----------- ---------- ----------
10                     19          0          6
2
@/u01/oracle/rdbms/admin/utlchain.sql
analyze table scott.t1 LIST CHAINED ROWS;
create table scott.t2 as select * from scott.t1 where rowid in (select HEAD_ROWID from chained_rows);
delete table scott.t1 where rowid in (select HEAD_ROWID from chained_rows);
insert into scott.t1 select * from sott.t2;
drop table scott.t2;
```

- ##### 行链接

```
SCOTT@ prod>create table t1 (c1 varchar2(4000),c2 varchar2(4000));
SCOTT@ prod>insert into t1 values(lpad('a',4000,'*'),lpad('b',4000,'*'));
SCOTT@ prod>commit;
SCOTT@ prod>analyze table t1 compute statistics;
SCOTT@ prod>select table_name, AVG_ROW_LEN,CHAIN_CNT from user_tables where table_name='T1';
TABLE_NAME                     AVG_ROW_LEN  CHAIN_CNT
------------------------------ ----------- ----------
T1                                    8015          1
SYS@ prod>create tablespace ttt datafile '/u01/oradata/prod/ttt01.dbf' size 10m blocksize 16k;
SCOTT@ prod>alter table t1 move tablespace ttt;
SCOTT@ prod>analyze table t1 compute statistics;
SCOTT@ prod>select table_name, AVG_ROW_LEN,CHAIN_CNT from user_tables where table_name='T1';
TABLE_NAME                     AVG_ROW_LEN  CHAIN_CNT
------------------------------ ----------- ----------
T1                                    8009          0
```

##### shrink

```
alter table t2 enable row movement;  
alter table t2 shrink space compact;
analyze table t2 compute statistics for table;
select table_name, blocks, empty_blocks, num_rows from user_tables where table_name='T2';
alter table t2 shrink space;
analyze table t2 compute statistics;
select table_name,blocks,empty_blocks,num_rows from user_tables where table_name='T2';
TABLE_NAME                         BLOCKS EMPTY_BLOCKS   NUM_ROWS
------------------------------ ---------- ------------ ----------
T2                                    426           22      28875
```



### 八、临时表空间

```
SQL> select file_id,file_name,tablespace_name from dba_temp_files;
SQL> select file#,name ,bytes/1024/1024 from v$tempfile;
SQL> create temporary tablespace temp2 tempfile '/u01/oradata/prod/temp02.dbf' size 10m;
SQL> alter tablespace temp2 add tempfile '/u01/oradata/prod/temp03.dbf' size 5m;
SQL> select file_id,file_name,tablespace_name from dba_temp_files;
FILE_ID FILE_NAME                                TABLESPACE_NAME
---------- -------------------------------------------------------------------------------- -----------------------
1     /u01/oradata/prod/temp01.dbf                     TEMP
2     /u01/oradata/prod/temp02.dbf                     TEMP2
3     /u01/oradata/prod/temp03.dbf                     TEMP2

SQL> alter tablespace temp2 drop tempfile '/u01/oradata/prod/temp03.dbf';
SQL> select file_id,file_name,tablespace_name from dba_temp_files;
SQL> select * from database_properties where rownum<=5;
SQL> alter user scott temporary tablespace temp2;  
SQL> alter database default temporary tablespace temp2;
```

##### 临时表空间组

```
SQL> alter tablespace temp1 tablespace group tmpgrp;
SQL> alter tablespace temp2 tablespace group tmpgrp;
SQL> select * from dba_tablespace_groups;
GROUP_NAME                     TABLESPACE_NAME
------------------------------ ------------------------------
TMPGRP                          TEMP1
TMPGRP                          TEMP2

SQL> alter database default temporary tablespace tmpgrp;
SQL> alter database default temporary tablespace temp;
SQL> alter tablespace temp1 tablespace group '';
SQL> alter tablespace temp2 tablespace group '';

SQL> select * from dba_tablespace_groups;
no rows selected
SQL> drop tablespace temp2 including contents and datafiles;
```

##### 扩充表空间的三种方法

```
create tablespace prod datafile '/u01/oradata/prod/prod01.dbf' size 5m;
alter tablespace prod add datafile '/u01/oradata/prod/prod02.dbf' size 20m;
alter database datafile '/u01/oradata/prod/prod01.dbf' autoextend on next 10m maxsize 500m;
```

##### 预防session挂起机制

```
alter session enable resumable;
select session_id,sql_text,error_number from dba_resumable;
select sid,event,seconds_in_wait from v$session_wait where sid=136;
alter session disable resumable;
```



### 九、分区表相关

##### 创建范围分区表

```
create table sale(
product_id varchar2(5), sales_count number(10,2))
partition by range(sales_count)
(
  partition p1 values less than(1000),
  partition p2 values less than(2000),
  partition p3 values less than(3000)
);
```

##### 增加一个分区

```
alter table sale add partition p4 values less than(maxvalue);
```

##### 查看分区基本面的视图

```
SQL>select * from user_part_key_columns where name='SALE';
```

##### 看一下段的分配

```
SQL> select segment_name,segment_type,partition_name from user_segments;
```

##### 删去分区

```
SQL>alter table sale drop partition p2;  
```

##### 查分区表分区数

```
select count(*) 
from user_tab_partitions TT where TT.table_name='TMS_MAIL_POSTING_INFO_SEC' 
and to_date(substr(func_longtovarchar2(TT.table_name,TT.partition_name),10,20),'YYYY-MM-DD HH24:MI:SS')<TO_DATE('2020-01-02', 'YYYY-MM-DD HH24:MI:SS')
order by to_date(substr(func_longtovarchar2(TT.table_name,TT.partition_name),10,20)) desc;
```

##### 批量删分区sql

```
select 'alter table ' || TT.table_name ||' drop partition ' || TT.partition_name || ';' 
from user_tab_partitions TT where TT.table_name='TMS_MAIL_POSTING_INFO_SEC' 
and to_date(substr(func_longtovarchar2(TT.table_name,TT.partition_name),10,20),'YYYY-MM-DD HH24:MI:SS')<TO_DATE('2020-01-02', 'YYYY-MM-DD HH24:MI:SS')
order by to_date(substr(func_longtovarchar2(TT.table_name,TT.partition_name),10,20)) desc;



select /*+first_rows*/* from tms_mail_posting_info partition (SYS_P11509);
```

##### 插入分区

```
alter table sale split partition p3 at (2000) into (partition p2,partition p3);
```

##### 合并分区

```
alter table sale merge partitions p2,p3 into partition p3;
```

##### 拆分分区

```
alter table sale split partition p3 at (2000) into (partition p2,partition p3);
```

##### 分区表update操作

```
alter table sale enable row movement;
update sale set sales_count=1200 where sales_count=600;
select rowid,t1.* from sale partition(p2) t1;
```

##### 分区表全局索引与分区索引

```
select * from user_ind_partitions;
create index sale_idx on sale(sales_count) local;
create index sale_global_idx on sale(sales_count) global
partition by range (sales_count)
(
partition p1 values less than(1500),
partition p2 values less than(maxvalue)
);
SQL>alter table sale drop partition p2 update indexes; 
SQL>alter index SALE_GLOBAL_IDX rebuild partition p1;
```

##### Hash分区

```
create table my_emp(empno number, ename varchar2(10))
partition by hash(ename) 
(
  partition p1, partition p2
);
```

##### list分区

```
create table personcity(id number, name varchar2(10), city varchar2(10))
partition by list(city)
(
  partition east values('tianjin','dalian'),
  partition west values('xian'),
  partition south values ('shanghai'),
  partition north values ('herbin'),
  partition other values (default)
);
SQL>select TABLE_NAME,PARTITION_NAME,HIGH_VALUE from user_tab_partitions where table_name='PERSONCITY';
SQL>select TABLE_NAME,PARTITIONING_TYPE,PARTITION_COUNT,STATUS from USER_PART_TABLES where TABLE_NAME='PERSONCITY';
select * from personcity partition(east);
```

复合分区

```
create table student(sno number, sname varchar2(10))
partition by range(sno)
subpartition by hash(sname)
subpartitions 4
(
  partition p1 values less than(1000),
  partition p2 values less than(2000),
  partition p3 values less than(maxvalue)
);

SQL> select TABLE_NAME,PARTITION_NAME,SUBPARTITION_COUNT,HIGH_VALUE 
from user_tab_partitions where TABLE_NAME='STUDENT';
SQL> select TABLE_NAME,PARTITION_NAME,SUBPARTITION_NAME from user_tab_subpartitions where table_name='STUDENT';
```

##### 分区重命名

```
alter table INTERVAL_SALES rename partition SYS_P33 to p2;
```

##### 创建不同类型分区表

```
CREATE TABLE purchase_orders 
  (po_id NUMBER(4),
   po_date TIMESTAMP, 
   supplier_id NUMBER(6), 
   po_total NUMBER(8,2),
   CONSTRAINT order_pk PRIMARY KEY(po_id)) 
PARTITION BY RANGE(po_date)
  (PARTITION Q1 VALUES LESS THAN (TO_DATE('2007-04-01','yyyy-mm-dd')), 
   PARTITION Q2 VALUES LESS THAN (TO_DATE('2007-06-01','yyyy-mm-dd')), 
   PARTITION Q3 VALUES LESS THAN (TO_DATE('2007-10-01','yyyy-mm-dd')), 
   PARTITION Q4 VALUES LESS THAN (TO_DATE('2008-01-01','yyyy-mm-dd')));
SQL>
CREATE TABLE purchase_order_items 
  (po_id NUMBER(4) NOT NULL, 
   product_id NUMBER(6) NOT NULL, 
   unit_price NUMBER(8,2), 
   quantity NUMBER(8), 
   CONSTRAINT po_items_fk FOREIGN KEY (po_id) REFERENCES purchase_orders(po_id)) 
   PARTITION BY REFERENCE(po_items_fk);

SQL> select TABLE_NAME,PARTITION_NAME,HIGH_VALUE from user_tab_partitions;
SQL> select TABLE_NAME,PARTITIONING_TYPE,REF_PTN_CONSTRAINT_NAME from user_part_tables;

create table emp1
  (empno number(4) primary key,
   ename char(10) not null,
   salary number(5) not null,
   bonus number(5)  not null,
   total_sal AS (salary+bonus))
partition by range (total_sal)
  (partition p1 values less than (5000),
   partition p2 values less than (maxvalue))
   enable row movement;
```

##### 增加约束

```
SQL>alter table scott.emp1 add constraint pk_emp1 primary key(empno);
SYS>alter table scott.emp1 add constraint chk_emp1 check (sal>500);
```

##### 创建物化视图

```
1）检查原始表是否具有在线重定义资格，（要求表自包含及之前没有建立实体化视图及日志）
SQL>
BEGIN
  DBMS_REDEFINITION.CAN_REDEF_TABLE('scott','emp1');
END;
/
CREATE TABLE scott.emp1_temp
  (empno        number(4) not null,
   ename        varchar2(10),
   job          varchar2(9),
   mgr          number(4),
   hiredate     date,
   sal          number(7,2),
   deptno       number(2))
PARTITION BY RANGE(sal)
   (PARTITION sal_low VALUES LESS THAN(2500),
   PARTITION sal_high VALUES LESS THAN (maxvalue));

SQL>
BEGIN
  dbms_redefinition.start_redef_table('scott','emp1','emp1_temp',
   'empno empno,
   ename ename,
   job job,
   mgr mgr,
   hiredate hiredate,
   sal sal,
   deptno deptno');
END;
/
SQL> select count(*) from scott.emp1_temp;
  COUNT(*)
----------
        14
SQL> select * from scott.emp1_temp partition(sal_low);
SQL> select * from scott.emp1_temp partition(sal_high);
这个时候emp1_temp的主键、索引、触发器和授权等还没有从原始表继承过来。
SQL> select table_name,constraint_name from user_constraints where table_name='EMP1_TEMP';
TABLE_NAME                     CONSTRAINT_NAME
------------------------------ ------------------------------
EMP1_TEMP                      SYS_C0011111             这是非空约束
SQL>select table_name,index_name from user_indexes where table_name='EMP1_TEMP';
no rows selected                                    没有索引


SQL>
DECLARE
  log_error int;
BEGIN
  DBMS_REDEFINITION.COPY_TABLE_DEPENDENTS(
uname   =>'SCOTT',
orig_table  =>'EMP1',
int_table  =>'EMP1_TEMP',
num_errors =>log_error
);
END;
/

SQL> select table_name,constraint_name,SEARCH_CONDITION from user_constraints where table_name='EMP1_TEMP';
TABLE_NAME                     CONSTRAINT_NAME             SEARCH_CONDITION
------------------------------ --------------------    -----------------------
EMP1_TEMP                      SYS_C0011111               "EMPNO" IS NOT NULL

SQL>select table_name,index_name from user_indexes where table_name='EMP1_TEMP';

5) 完成重定义过程。
SQL> EXECUTE dbms_redefinition.finish_redef_table('scott','emp1','emp1_temp');
SQL> select table_name,partition_name,high_value from user_tab_partitions;
SQL>select table_name,constraint_name from user_constraints where table_name='EMP1';
SQL>select table_name,index_name from user_indexes where table_name='EMP1';
```

##### 创建索引组织表

```
SQL> create table iot_prod(id int, name char(50), sal int, hiredate date, constraint pk_prod primary key (id)) organization index pctthreshold 30 overflow including sal tablespace test;

SQL> select tablespace_name,segment_name,segment_type,partition_name from user_segments;
```

##### 创建簇表

```
SQL> create cluster cluster1(code_key number);
SQL> create table emp1 (empno int,ename char(10),deptno number) cluster cluster1(deptno);
SQL> create table dept1 (deptno number,loc char(10)) cluster cluster1(deptno);
SQL> create index index1 on cluster cluster1;
生成了cluster1段和index1段。
SQL>select dbms_rowid.ROWID_RELATIVE_FNO(rowid) fn, dbms_rowid.rowid_block_number(rowid) bl, rowid,deptno 
from emp1;

SQL>select dbms_rowid.ROWID_RELATIVE_FNO(rowid) fn, dbms_rowid.rowid_block_number(rowid) bl, rowid,deptno from dept1;
```

##### 删除簇表

```
drop table emp1
drop table detp1;
drop cluster cluster1;  最后删除簇
analyze table emp1 compute statistics;
```

##### 得到当前session的信息

```
select sid from v$mystat where rownum=1;   

select sid,server,status from v$session where sid=46;
```

##### 外部表示例

```
select empno||','||ename||','||sal||','||deptno from scott.emp;
$mkdir -p /home/oracle/dir1
$vi /home/oracle/dir1/emp1.dat   粘贴查询结果
create directory dir1 as '/home/oracle/dir1'; 
grant read,write on directory dir1 to scott,tim;
CREATE TABLE emp1_ext  
    (EMPNO      NUMBER(4),
     ENAME       VARCHAR2(10),
     SAL             NUMBER(7,2),
     DEPTNO     NUMBER(2))
  ORGANIZATION EXTERNAL
     (TYPE ORACLE_LOADER 
      DEFAULT DIRECTORY dir1
        ACCESS PARAMETERS (FIELDS TERMINATED BY ",")
          LOCATION ('emp1.dat')
            ) REJECT LIMIT UNLIMITED;
```

##### ORACLE_DATAPUMP引擎导出外部表

```
SQL> CREATE TABLE emp2_ext 
ORGANIZATION EXTERNAL
( TYPE ORACLE_DATAPUMP  
  DEFAULT DIRECTORY dir1
  LOCATION ('emp2.dmp')) 
AS SELECT empno,ename,sal,deptno FROM scott.emp ;
```

##### ORACLE_DATAPUMP引擎导入外部表

```
SQL> CREATE TABLE emp3_ext 
   (EMPNO   NUMBER(4), 
    ENAME   VARCHAR2(10),
    SAL  NUMBER(7,2),
    DEPTNO NUMBER(2)) 
ORGANIZATION EXTERNAL   
(  
  TYPE ORACLE_DATAPUMP 
  DEFAULT DIRECTORY dir1 
  LOCATION ('emp2.dmp')   
) ;
```

##### 创建物化视图

```
create table test (id int,name char(10));
create materialized view log on test with rowid;
conn / as sysdba
grant create materialized view to scott;
create materialized view test_view1 refresh fast with rowid on commit enable query rewrite as select * from test;

drop materialized view test_view1;
drop materialized view log on test;
create materialized view log on test with rowid;

select * from tab;
select * from tab@my_link;
grant create materialized view to scott;
create materialized view test_view2 refresh fast  
start with sysdate next sysdate+1/2880
with rowid as select * from scott.test@my_link;
create materialized view test_view3 refresh fast
with rowid as select * from scott.test@my_link;

create table emp1 as select * from emp;
create materialized view log on emp1 with rowid,sequence(sal,deptno) including new values;
create materialized view emp1_view1 refresh start with sysdate next sysdate+1/2880 as select deptno,sum(sal) sumsal from emp1 group by deptno;
create materialized view emp1_view2 refresh with rowid on demand 
as select deptno,sum(sal) sumsal from emp1@my_link  group by deptno;
exec dbms_mview.refresh('emp1_view2','?');
```