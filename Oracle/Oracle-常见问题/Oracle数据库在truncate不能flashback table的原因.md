---
title: oracle数据库在truncate不能flashback table的原因
date: 2019-9-22
categories: Oracle
tags: Oracle常见问题
---



#

#### 原因解释：

​	Commend：
​        truncate 命令改变了data_object_id，即实际的id
​        delete   命令没有改变了data_object_id
​        OBJECT_ID和DATA_OBJECT_ID同样是表示数据库对象的唯一标示，但是OBJECT_ID标示的是逻辑id，DATA_OBJECT_ID标示的物理id。
​        flashback table是基于数据库的undo表空间，即：flashback是从undo中获取数据( undo中的数据在commit之后随时可能会被覆盖)。如果是truncate的table.从dba_objects中可以看到，truncate之后的table的data_objcet_id发生了改变，oracle无法得知原来的block id,即：oracle无法在物理上找到原来表的位置对其进行恢复。

#### 实验论证步骤如下：

##### 实验一：  

```shell
SYS@prod>select object_name,object_id,data_object_id from dba_objects where owner='SCOTT';

OBJECT_NAME	      OBJECT_ID DATA_OBJECT_ID
-------------------- ---------- --------------
PK_DEPT 		  87107 	 87107
DEPT			  87106 	 87106
EMP			  87108 	 87108
PK_EMP			  87109 	 87109
BONUS			  87110 	 87110
SALGRADE		  87111 	 87111

已选择6行。

SYS@prod>create table scott.t1(id int);

表已创建。

SYS@prod>insert into scott.t1 values(10000);

已创建 1 行。

SYS@prod>commit;

提交完成。

SYS@prod>select * from scott.t1;

	ID
----------
     10000

SYS@prod>select object_name,object_id,data_object_id from dba_objects where owner='SCOTT';

OBJECT_NAME	      OBJECT_ID DATA_OBJECT_ID
-------------------- ---------- --------------
PK_DEPT 		  87107 	 87107
DEPT			  87106 	 87106
EMP			  87108 	 87108
PK_EMP			  87109 	 87109
BONUS			  87110 	 87110
SALGRADE		  87111 	 87111
T1			  88961 	 88961

已选择7行。

SYS@prod>select current_scn from v$database;

CURRENT_SCN
-----------
    1386775

SYS@prod>truncate table scott.t1;

表被截断。

SYS@prod>alter table scott.t1 enable row movement;

表已更改。

SYS@prod>flashback table scott.t1 to scn 1386775;
flashback table scott.t1 to scn 1386775
                      *
第 1 行出现错误:
ORA-01466: 无法读取数据 - 表定义已更改


SYS@prod>select object_name,object_id,data_object_id from dba_objects where owner='SCOTT';

OBJECT_NAME	      OBJECT_ID DATA_OBJECT_ID
-------------------- ---------- --------------
PK_DEPT 		  87107 	 87107
DEPT			  87106 	 87106
EMP			  87108 	 87108
PK_EMP			  87109 	 87109
BONUS			  87110 	 87110
SALGRADE		  87111 	 87111
T1			  88961 	 88962

已选择7行

实验二：
SYS@prod>create table scott.t1(id int);

表已创建。

SYS@prod>insert into scott.t1 values(10000);

已创建 1 行。

SYS@prod>commit;

提交完成。

SYS@prod>select * from scott.t1;

	ID
----------
     10000

SYS@prod>select object_name,object_id,data_object_id from dba_objects where owner='SCOTT';

OBJECT_NAME	      OBJECT_ID DATA_OBJECT_ID
-------------------- ---------- --------------
PK_DEPT 		  87107 	 87107
DEPT			  87106 	 87106
EMP			  87108 	 87108
PK_EMP			  87109 	 87109
BONUS			  87110 	 87110
SALGRADE		  87111 	 87111
T1			  88967 	 88967

已选择7行。

SYS@prod>select current_scn from v$database;

CURRENT_SCN
-----------
    1387676

SYS@prod>delete scott.t1;

已删除 1 行。

SYS@prod>commit;
提交完成。

SYS@prod>select object_name,object_id,data_object_id from dba_objects where owner='SCOTT';

OBJECT_NAME	      OBJECT_ID DATA_OBJECT_ID
-------------------- ---------- --------------
PK_DEPT 		  87107 	 87107
DEPT			  87106 	 87106
EMP			  87108 	 87108
PK_EMP			  87109 	 87109
BONUS			  87110 	 87110
SALGRADE		  87111 	 87111
T1			  88967 	 88967

已选择7行。

SYS@prod>alter table scott.t1 enable row movement;

表已更改。

SYS@prod>flashback table scott.t1 to scn 1387676;

闪回完成。

SYS@prod>select * from scott.t1;

	ID
----------
     10000
```