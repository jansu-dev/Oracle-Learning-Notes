---
title: 批量插入操作实验  之【Archivelog-NoLogging-Append-NoPartition】
date: 2019-11-03
---




##### 1.实验运行脚本前：
```
SYS@prod>archive log list
Database log mode        Archive Mode
Automatic archival         Enabled
Archive destination        /u01/arch
Oldest online log sequence     14208
Next log sequence to archive   14210
Current log sequence           14210
```
数据库为归档模式

```
SYS@prod>select name,value from v$sysstat where name='redo size';

NAME          VALUE
---------- ----------
redo size     84124
```
在实验前查询归档日志大小

```
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25305628   1802164  94% /
tmpfs             961096    92816    868280  10% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
[oracle@bulkinsert arch]$ ll
total 0
```
实验前查看一下硬盘使用情况。



##### 2.实验SQL插入【2千万行】
```
SCOTT@prod>set timing on
SCOTT@prod>create table bulk8 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:00.99
SCOTT@prod>insert /*+ append */ into bulk8 nologging select * from bulkinsert;
```



##### 3.实验结果
```
SCOTT@prod>insert /*+ append */ into bulk8 nologging select * from bulkinsert;

20000000 rows created.

Elapsed: 00:06:51.72
```


##### 4.运行脚本后：   
```
SYS@prod>col name for a20
SYS@prod>select name,value from v$sysstat where name='redo size';

NAME          VALUE
---------- ----------
redo size  1319071276
```


```  
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 22759484   4348308  84% /
tmpfs             961096   479164    481932  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot  

[oracle@bulkinsert arch]$ cd /u01/arch/
[oracle@bulkinsert arch]$ ll
total 6431048
-rw-r-----. 1 oracle oinstall 1380864 Oct 24 23:52 1_1576_1022509032.log
-rw-r-----. 1 oracle oinstall 5109248 Oct 25 00:03 1_1577_1022509032.log
......
......

[oracle@bulkinsert arch]$ rm *
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 16369596  10738196  61% /
tmpfs             961096   479164    481932  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
```



