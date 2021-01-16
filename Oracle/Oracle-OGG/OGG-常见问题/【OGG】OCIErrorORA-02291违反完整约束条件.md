---
title: 【OGG】解决OCIErrorORA-02291:违反完整约束条件错误问题 
date: 2019-11-1 
categories: Oracle
tags: Oracle-OGG
---



### OGG双向连通问题描述：   

在做OGG双向连通的时候，主库端的（replicat）复制进程rora_1总是启动失败。   

```  
GGSCI (ogg1) 147> info all

Program     Status      Group       Lag at Chkpt  Time Since Chkpt

MANAGER     RUNNING                                           
EXTRACT     RUNNING     EORA_1      00:00:00      00:00:10    
EXTRACT     RUNNING     PORA_1      00:00:00      00:00:03    
REPLICAT    ABENDED     RORA_1      00:00:06      00:05:24    
```


### view report rora_1查看报告：   

```
GGSCI (ogg1) 146> view report rora_1

***********************************************************************
                 Oracle GoldenGate Delivery for Oracle
    Version 11.2.1.0.1 OGGCORE_11.2.1.0.1_PLATFORMS_120423.0230_FBO
   Linux, x64, 64bit (optimized), Oracle 11g on Apr 23 2012 08:48:07
 
Copyright (C) 1995, 2012, Oracle and/or its affiliates. All rights reserved.

         ..............
         ..省略部分信息..
         ..............

Wildcard MAP resolved (entry scott.*):
  MAP "SCOTT"."EMP", TARGET scott."EMP";
Using following columns in default map by name:
  EMPNO, ENAME, JOB, MGR, HIREDATE, SAL, COMM, DEPTNO
Using the following key columns for target table SCOTT.EMP: EMPNO.

2019-10-31 17:58:49  WARNING OGG-00869  OCI Error ORA-02291: 违反完整约束条
件 (SCOTT.FK_DEPTNO) - 未找到父项关键字 (status = 2291). INSERT INTO "SCOTT"
."EMP" ("EMPNO","ENAME","JOB","MGR","HIREDATE","SAL","COMM","DEPTNO") VALUES
 (:a0,:a1,:a2,:a3,:a4,:a5,:a6,:a7).

2019-10-31 17:58:49  WARNING OGG-01004  Aborted grouped transaction on 'SCOT
T.EMP', Database error 2291 (OCI Error ORA-02291: 违反完整约束条件 (SCOTT.FK
_DEPTNO) - 未找到父项关键字 (status = 2291). INSERT INTO "SCOTT"."EMP" ("EMP
NO","ENAME","JOB","MGR","HIREDATE","SAL","COMM","DEPTNO") VALUES (:a0,:a1,:a
2,:a3,:a4,:a5,:a6,:a7)).

2019-10-31 17:58:49  WARNING OGG-01003  Repositioning to rba 1154 in seqno 2
.

...skipping one line

2019-10-31 17:58:49  WARNING OGG-01003  Repositioning to rba 1154 in seqno 2
.

Source Context :
  SourceModule            : [er.errors]
  SourceID                : [/scratch/aime1/adestore/views/aime1_adc4150256/
oggcore/OpenSys/src/app/er/errors.cpp]
  SourceFunction          : [take_rep_err_action]
  SourceLine              : [623]
  ThreadBacktrace         : [8] elements
                          : [/u01/app/ogg/libgglog.so(CMessageContext::AddTh
readContext()+0x1e) [0x7f7b8d99b06e]]
                          : [/u01/app/ogg/libgglog.so(CMessageFactory::Creat
eMessage(CSourceContext*, unsigned int, ...)+0x2cc) [0x7f7b8d99744c]]
                          : [/u01/app/ogg/libgglog.so(_MSG_ERR_MAP_TO_TANDEM
_FAILED(CSourceContext*, ggs::gglib::ggapp::CQualDBObjName<(DBObjType)1> con
st&, ggs::gglib::ggapp::CQualDBObjName<(DBObjType)1> const&, CMessageFactory
::MessageDisposition)+0x53) [0x7f7b8d98ff19]]
                          : [/u01/app/ogg/replicat(take_rep_err_action(short
, int, char const*, extr_ptr_def*, __std_rec_hdr*, char*, file_def*, bool)+0
xdac) [0x51daa0]]
                          : [/u01/app/ogg/replicat(process_extract_loop()+0x
2240) [0x536ab0]]
                          : [/u01/app/ogg/replicat(main+0x732) [0x548752]]
                          : [/lib64/libc.so.6(__libc_start_main+0xfd) [0x3cc
321ed1d]]
                          : [/u01/app/ogg/replicat(__gxx_personality_v0+0x32
2) [0x4be48a]]

2019-10-31 17:58:49  ERROR   OGG-01296  Error mapping from SCOTT.EMP to SCOT
T.EMP.

***********************************************************************
*                   ** Run Time Statistics **                         *
***********************************************************************

         ..............
         ..省略部分信息..
         ..............


2019-10-31 17:58:49  ERROR   OGG-01668  PROCESS ABENDING.

CACHE OBJECT MANAGER statistics

CACHE MANAGER VM USAGE
vm current     =      0    vm anon queues =      0 
vm anon in use =      0    vm file        =      0 
vm used max    =      0    ==> CACHE BALANCED

CACHE CONFIGURATION
cache size       =   2G   cache force paging = 3.41G
buffer min       =  64K   buffer highwater   =   8M
pageout eligible size =   8M

============================================================================
====
RUNTIME STATS FOR SUPERPOOL

CACHE Transaction Stats
trans active   =      0    max concurrent =      0 
non-zero total =      0    trans total    =      0 
```


### 错误信息提示   

##### 提示： 

- OCI Error ORA-02291: 违反完整约束条件 (SCOTT.FK_DEPTNO) - 未找到父项关键字 (status = 2291).   INSERT INTO "SCOTT""EMP" ......  

##### 可知：  

- 在错误报告中可以查看到错误报告，给出的提示是OGG的RORA_1进程在做插入操作的时候，违反了完整性约束。   
- 于是间，恍然大悟之前在做操作主库到备库连接操作的时候，主库做了delete scott.dept;操作，但是主库的操作并没有被传到备库端，所以造成了主库的dept表没有任何记录仅有表结构，但是备库中的dept表中记录是全的。
- 紧接着在主库的raplicat进程没开启的情况下，为了测试备库操作是否能够同步到主库，在备库端的scott.emp表操作了insert，最终导致了主库端的rora_1进程一直起不来。
- 【!-感叹-!】：在不清楚错误之前，一定不要轻易做操作，否则可能引发一系列的连环错误操作。



### 解决办法一、将主库的约束(constraint)失效
##### 主库端：
```
SYS@prod>select constraint_name from dba_constraints where owner='SCOTT';

CONSTRAINT_NAME
------------------------------
FK_DEPTNO
PK_DEPT
PK_EMP

SYS@prod>alter table scott.emp disable constraint FK_DEPTNO;
Table altered.   
```
##### 主库端：  
```
GGSCI (ogg1) 10> start rora_1
Sending START request to MANAGER ...
REPLICAT RORA_1 starting


GGSCI (ogg1) 11> info all
Program     Status      Group       Lag at Chkpt  Time Since Chkpt

MANAGER     RUNNING                                           
EXTRACT     RUNNING     EORA_1      00:00:00      00:00:03    
EXTRACT     RUNNING     PORA_1      00:00:00      00:00:00    
REPLICAT    RUNNING     RORA_1      00:00:00      00:00:00    

```
##### 主库端：
```
SYS@prod>alter table scott.emp enable constraint FK_DEPTNO;

Table altered.   
```
最后在开启constraint约束。

##### 决绝问题
*  至此问题解决，但是此时便会造成主备库之间信息不一致的现象。
*  我们可在开始采用解决方法二，以imp方式将主库中的emp和dept两个表重新导入



### 解决办法二、imp/emp方式补全两个表   

##### 主库端
```
SCOTT@prod>drop table emp;
Table dropped.

SCOTT@prod>drop table dept;
Table dropped.
```
我采用导出用户的方式实现将表补全，先删除两个表之后才能导表，否者会出现错误。   
接下来进行导表操作，并验证表是否真正的被重新导入。

```
[oracle@ogg1 ~]$ imp scott/scott@bj file=scott.dmp fromuser=scott touser=scott

Import: Release 11.2.0.4.0 - Production on 星期五 11月 1 13:18:37 2019
Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
Export file created by EXPORT:V11.02.00 via conventional path
Warning: the objects were exported by SYS, not by you

import done in AL32UTF8 character set and AL16UTF16 NCHAR character set
IMP-00015: following statement failed because the object already exists:
 "CREATE TABLE "BONUS" ("ENAME" VARCHAR2(10), "JOB" VARCHAR2(9), "SAL" NUMBER"
 ", "COMM" NUMBER)  PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255            "
 "        LOGGING NOCOMPRESS"
. . importing table                         "DEPT"          4 rows imported
. . importing table                          "EMP"         14 rows imported
IMP-00015: following statement failed because the object already exists:
 "CREATE TABLE "SALGRADE" ("GRADE" NUMBER, "LOSAL" NUMBER, "HISAL" NUMBER)  P"
 "CTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255 STORAGE(INITIAL 65536 NEXT 104"
 "8576 MINEXTENTS 1 FREELISTS 1 FREELIST GROUPS 1 BUFFER_POOL DEFAULT)       "
 "             LOGGING NOCOMPRESS"
About to enable constraints...
Import terminated successfully with warnings.

[oracle@ogg1 ~]$ sqlplus / as sysdba
SQL*Plus: Release 11.2.0.4.0 Production on 星期五 11月 1 13:18:51 2019
Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prod>select * from scott.dept;

    DEPTNO DNAME    LOC
---------- -------------- -------------
  10        ACCOUNTING   NEW YORK
  20        RESEARCH     DALLAS
  30        SALES        CHICAGO
  40        OPERATIONS   BOSTON

SYS@prod>select * from scott.emp;

     EMPNO ENAME      JOB        MGR HIREDATE         SAL COMM   DEPTNO
---------- ---------- --------- ---------- ------------------- ---------- ---------- ----------
      7369 SMITH      CLERK       7902 1980-12-17 00:00:00        800      20
      7499 ALLEN      SALESMAN        7698 1981-02-20 00:00:00       1600  300       30
      7521 WARD       SALESMAN        7698 1981-02-22 00:00:00       1250  500       30
      7566 JONES      MANAGER       7839 1981-04-02 00:00:00       2975      20
      7654 MARTIN     SALESMAN        7698 1981-09-28 00:00:00       1250 1400       30
      7698 BLAKE      MANAGER       7839 1981-05-01 00:00:00       2850      30
      7782 CLARK      MANAGER       7839 1981-06-09 00:00:00       2450      10
      7788 SCOTT      ANALYST       7566 1987-04-19 00:00:00       3000      20
      7839 KING       PRESIDENT      1981-11-17 00:00:00       5000      10
      7844 TURNER     SALESMAN        7698 1981-09-08 00:00:00       1500    0       30
      7876 ADAMS      CLERK       7788 1987-05-23 00:00:00       1100      20
      7900 JAMES      CLERK       7698 1981-12-03 00:00:00        950      30
      7902 FORD       ANALYST       7566 1981-12-03 00:00:00       3000      20
      7934 MILLER     CLERK       7782 1982-01-23 00:00:00       1300      10

14 rows selected.
```

##### 主库端ggsci中操作   
```
GGSCI (ogg1) 8> info all

Program     Status      Group       Lag at Chkpt  Time Since Chkpt

MANAGER     RUNNING                                           
EXTRACT     RUNNING     EORA_1      16:30:44      00:00:09    
EXTRACT     RUNNING     PORA_1      00:00:00      16:30:31    
REPLICAT    ABENDED     RORA_1      00:00:00      16:30:25    


GGSCI (ogg1) 9> start rora_1

Sending START request to MANAGER ...
REPLICAT RORA_1 starting


GGSCI (ogg1) 10> info all

Program     Status      Group       Lag at Chkpt  Time Since Chkpt

MANAGER     RUNNING                                           
EXTRACT     RUNNING     EORA_1      00:00:00      00:00:03    
EXTRACT     RUNNING     PORA_1      00:00:00      00:00:00    
REPLICAT    RUNNING     RORA_1      00:00:00      00:00:00    

```
开启主库端的RORA_1进程


##### 备库端
```
SYS@prod>select * from scott.dept;

    DEPTNO     DNAME          LOC
---------- -------------- -------------
  10        ACCOUNTING    NEW YORK
  20        RESEARCH      DALLAS
  30        SALES         CHICAGO
  40        OPERATIONS    BOSTON


SYS@prod>insert into scott.dept values(50,'BOSS','SY');

1 row created.

SYS@prod>commit;

Commit complete.

SYS@prod>select * from scott.dept;

    DEPTNO    DNAME         LOC
---------- -------------- -------------
  50        BOSS            SY
  10        ACCOUNTING      NEW YORK
  20        RESEARCH        DALLAS
  30        SALES           CHICAGO
  40        OPERATIONS      BOSTON 

```
在备库端做查库操作，已验证备库到主库是否连通。insert into scott.dept values(50,'BOSS','SY');




##### 主库端
```
SYS@prod>select * from scott.dept;

    DEPTNO DNAME    LOC
---------- -------------- -------------
  10        ACCOUNTING   NEW YORK
  20        RESEARCH     DALLAS
  30        SALES        CHICAGO
  40        OPERATIONS   BOSTON


         ...........................
         ..需要多刷新几遍，可能存在延迟..
         ...........................



SYS@prod>select * from scott.dept;

    DEPTNO    DNAME         LOC
---------- -------------- -------------
  50        BOSS            SY
  10        ACCOUNTING      NEW YORK
  20        RESEARCH        DALLAS
  30        SALES           CHICAGO
  40        OPERATIONS      BOSTON 

```



- 至此问题解决，备库端操作顺利被EORA_1进程捕获，PORA_1进程传送到主库端，并在主库端的RORA_1进程插入到表中双端连通问题解决。   
- 在实际生产库中不建议使表的完整性约束不可用这种方法，因为存在大量的业务，很可能会出现一些问题
- 当然使用imp的方法也很少用到，删表再重新导入是大忌，在生产库上很少会出现主备库不一致导致replicat进程无法开启现象。
- 在做双相连通的时候，一定要一步一步的做，如果出现报错信息一定要揪出其根本原因再往下做，否则可能造成更大的损失。























