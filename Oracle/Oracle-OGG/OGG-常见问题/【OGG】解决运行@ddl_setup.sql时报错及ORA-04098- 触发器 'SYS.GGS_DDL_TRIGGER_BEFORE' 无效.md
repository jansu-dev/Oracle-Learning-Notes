---
title: 【OGG】解决运行@ddl_setup.sql时报错问题  
date: 2019-10-28 
categories: Oracle
tags: Oracle-OGG
---



#### 一、版本信息  

```
Oracle版本信息：Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit
Linux版本信息：Linux ogg1 2.6.32-696.el6.x86_64 #1 SMP Tue Feb 21 00:53:17 EST 2017 x86_64 x86_64 x86_64 GNU/Linux  
OGG版本信息：Version 11.2.1.0.1 OGGCORE_11.2.1.0.1_PLATFORMS_120423.0230_FBO
```



#### 二、运行ddl_setup包报错信息如下：  

```
SYS@prod>@ddl_setup

Oracle GoldenGate DDL Replication setup script
......
......
NOTE: The schema must be created prior to running this script.
NOTE: Stop all DDL replication before starting this installation.

Enter Oracle GoldenGate schema name:ogg

Working, please wait ...
Spooling to file ddl_setup_spool.txt
Checking for sessions that are holding locks on Oracle Golden Gate metadata tables ...
The following sessions are holding 1 locks on objects owned by OGG :

  INST_ID  SID   SERIAL#  OS_USER  USERNAME  PID           PROGRAM
--------- ----- -------- -------- --------- -------- ------------------------
     1     41     289     oracle    SYS      4581    sqlplus@ogg1 (TNS V1-V3)

Details of locks being held:

   INST_ID    SID        SERIAL#    OBJECT_LOCKED   LOCK_TYPE
---------- ---------- ------------- -------------- ----------------
     1         41        289         GGS_STICK          TO


BEGIN
*
ERROR at line 1:
ORA-20783:
Oracle GoldenGate DDL Replication setup:
*** Disconnect all sessions that are holding locks on Oracle GoldenGate metadata tables, and retry.
ORA-06512: 在 line 3


Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
```



#### 三、解决方法：

此问题是版本导致的，本库Oracle版本为11.2.0.4.0，解决办法就是运行ddl_disable.sql包来禁用DDL触发器，之后在运行ddl_setup.sql包便可解决问题，最后再运行ddl_enable.sql包开启DDL触发器。  

<span style='color:red'>【注意】在11.1.1.2版本中，跑ddl_setup.sql包时需要手动输入 ogg,INITIALSETUP,yes</span>


##### 具体操作步骤

```
SYS@prod>@ddl_disable.sql

Trigger altered.

SYS@prod>@ddl_setup.sql

Oracle GoldenGate DDL Replication setup script
Verifying that current user has privileges to install DDL Replication...
You will be prompted for the name of a schema for the Oracle GoldenGate database objects.
NOTE: For an Oracle 10g source, the system recycle bin must be disabled. For Oracle 11g and later, it can be enabled.
NOTE: The schema must be created prior to running this script.
NOTE: Stop all DDL replication before starting this installation.

Enter Oracle GoldenGate schema name:ogg

Working, please wait ...
Spooling to file ddl_setup_spool.txt
Checking for sessions that are holding locks on Oracle Golden Gate metadata tables ...
Check complete.
Using OGG as a Oracle GoldenGate schema name.
Working, please wait ...

DDL replication setup script complete, running verification script...
Please enter the name of a schema for the GoldenGate database objects:
Setting schema name to OGG

CLEAR_TRACE STATUS:

Line/pos                 Error
----------------------- ---------------------
No errors                No errors


STATUS OF DDL REPLICATION
---------------------------------------------------------------
SUCCESSFUL installation of DDL Replication software components

Script complete.
```




#### 四、如果跨越OGG搭建步骤进行跑脚本可能会出现ORA-04098错误 

```
SQL> grant GGS_GGSUSER_ROLE to ogg;
grant GGS_GGSUSER_ROLE to ogg
*
ERROR at line 1:
ORA-04098: trigger 'SYS.GGS_DDL_TRIGGER_BEFORE' is invalid and failed re-validation
```



#### 五、解决方法：

* 出现了这个问题说明前面的包没有跑正确，尤其是ddl_setup包  
* ogg用户的一些权限必须以显示方式授权，直接个DBA权限是不能解决的。因此GGS_GGSUSER_ROLE角色需要单独授权。













