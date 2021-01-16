---
title: 批量插入操作实验  之【NoArchivelog-NoLogging-Append-NoPartition】
date: 2019-11-03
---

 


##### 1.实验运行脚本前：
```
SYS@prod>archive log list
Database log mode        No Archive Mode
Automatic archival         Disabled
Archive destination        /u01/arch
Oldest online log sequence     13377
Current log sequence           13379
```
数据库为非归档模式。

```
SYS@prod>col name for a10;
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME        VALUE
---------- ----------
redo size    72032
```
在实验前查询redo日志大小。

```
[oracle@bulkinsert ~]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 21821536   5286256  81% /
tmpfs             961096   479560    481536  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```
查看磁盘使用情况。



##### 2.实验SQL插入【2千万行】
```
SCOTT@prod>set timing on
SCOTT@prod>create table bulk3 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:02.19
SCOTT@prod>insert /*+ append */ into bulk3 nologging select * from scott.bulkinsert;
```



##### 3.实验结果
```
SCOTT@prod>insert /*+ append */ into bulk3 nologging select * from scott.bulkinsert;

20000000 rows created.

Elapsed: 00:01:14.60
```


##### 4.运行脚本后：   
```
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME            VALUE
----------   ----------
redo size      920864
```

```
[oracle@bulkinsert ~]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 22429628   4678164  83% /
tmpfs             961096   472032    489064  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```



