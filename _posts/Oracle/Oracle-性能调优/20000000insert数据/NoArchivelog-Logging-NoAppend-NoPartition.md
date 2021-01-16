---
title: 批量插入操作实验  之【NoArchivelog-Logging-NoAppend-NoPartition】
date: 2019-11-03
---




##### 1.实验运行脚本前：
```
SYS@prod>archive log list
Database log mode        No Archive Mode
Automatic archival         Disabled
Archive destination        /u01/arch
Oldest online log sequence     12835
Current log sequence           12837
```
数据库为非归档模式。

```
SYS@prod>col name for a10;
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME          VALUE
---------- ----------
redo size     9132592
```
在实验前查询归档日志大小。




##### 2.实验SQL插入【2千万行】
```shell
SCOTT@prod>set timing on
SCOTT@prod>create table bulk2 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:00.02
SCOTT@prod>insert into bulk1 select * from bulkinsert;
```


##### 3.实验结果
```
SCOTT@prod>insert into bulk1 select * from bulkinsert;

20000000 rows created.

Elapsed: 00:12:37.08
```


##### 4.运行脚本后：   
```
SYS@prod>col name for a10;
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME          VALUE
---------- ----------
redo size  1351534664
```
redo大小增加1342402072。

```
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 21820952   5286840  81% /
tmpfs             961096   480048    481048  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```
实验操作后查看一下硬盘使用情况。


