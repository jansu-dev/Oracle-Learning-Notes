---
title: 【错误】--ORA-27101.md
date: 2019:10:22  
categories: Oracle
tags: Oracle错误
---



### ORA-27101: shared memory realm does not exist


```shell
SYS@prod>shutdown abort
ORACLE 例程已经关闭。
SYS@prod>startup
ORACLE 例程已经启动。

Total System Global Area  154521600 bytes
Fixed Size          2251216 bytes
Variable Size         113247792 bytes
Database Buffers       33554432 bytes
Redo Buffers            5468160 bytes
数据库装载完毕。
ORA-03113: 通信通道的文件结尾 进程 ID:
3271
会话 ID: 1 序列号: 5

SYS@prod>exit
从 Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit
ProductionWith the Partitioning, OLAP, Data Mining and Real Application 
Testing options 断开
[oracle@oracle6 ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on 星期四 10月 10 04:21:50 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

ERROR:
ORA-09817: Write to audit file failed.
Linux-x86_64 Error: 28: No space left on device
Additional information: 12
ORA-09945: Unable to initialize the audit trail file
Linux-x86_64 Error: 28: No space left on device

请输入用户名:  sys
输入口令: 
ERROR:
ORA-01034: ORACLE not available
ORA-27101: shared memory realm does not exist
Linux-x86_64 Error: 2: No such file or directory
进程 ID: 0
会话 ID: 0 序列号: 0

```

[oracle@oracle6 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        28G   26G   17M 100% /
tmpfs           940M   72K  939M   1% /dev/shm
/dev/sda1       190M   39M  141M  22% /boot    

#### 此处可以看到在Linux层面磁盘空间的使用率已经达到了100%  
#### 因此没有共享的空间留给用户去正常开启数据库

###我的做法：
由于本数据库属于归档模式，故删除一部分归档，来腾出空间开启数据库  


```shell
cd /u01/oracle
[oracle@oracle6 oracle]$ cd ..
[oracle@oracle6 u01]$ ls
admin  cfgtoollogs  diag                oracle   oraInventory
arch   checkpoints  fast_recovery_area  oradata
[oracle@oracle6 u01]$ cd arch/
[oracle@oracle6 arch]$ ls
arch_1_1015435229_174.log    
arch_1_1015435229_233.log     
arch_1_1015435229_292.log
······
arch_1_1015435229_232.log    
arch_1_1015435229_291.log
[oracle@oracle6 arch]$ rm *
[oracle@oracle6 arch]$ ls
[oracle@oracle6 arch]$ 
[oracle@oracle6 arch]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        28G   18G  8.1G  69% /
tmpfs           940M   72K  939M   1% /dev/shm
/dev/sda1       190M   39M  141M  22% /boot   
[oracle@oracle6 u01]$ sqlplus / as sysdba  

SYS@prod>startup
ORACLE 例程已经启动。

Total System Global Area  154521600 bytes
Fixed Size          2251216 bytes
Variable Size         113247792 bytes
Database Buffers       33554432 bytes
Redo Buffers            5468160 bytes
数据库装载完毕。
数据库已经打开。
SYS@prod>select file_name,file_id,bytes/1024/1024||'M' from dba_data_files;

FILE_NAME                      FILE_ID BYTES/1024/1024||'M'
----------------------------- -----------------------------------------
/u01/oradata/prod/users01.dbf   4 1287.5M
/u01/oradata/prod/undotbs01.dbf 3 2280M
/u01/oradata/prod/sysaux01.dbf  2 730M
/u01/oradata/prod/system01.dbf  1 760M
/u01/oradata/prod/example01.dbf 5 345M
```


这样就可以正常打开数据库了。

