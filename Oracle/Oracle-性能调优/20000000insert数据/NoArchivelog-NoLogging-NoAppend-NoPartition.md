---
title: 批量插入操作实验  之【NoArchivelog-NoLogging-NoAppend-NoPartition】
date: 2019-11-02
---




##### 1.实验运行脚本前：
```
SYS@prod>archive log list
Database log mode          Archive Mode
Automatic archival         Enabled
Archive destination        /u01/arch
Oldest online log sequence     14474
Next log sequence to archive   14476
Current log sequence           14476
```
数据库为归档模式。
```
SYS@prod>col name for a10;
SYS@prod>select name,value from v$sysstat where name = 'redo size';

NAME        VALUE
---------- ----------
redo size   70816
```
实验前查看redo使用大小。
```
[oracle@bulkinsert arch]$ ll
total 0
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25306348   1801444  94% /
tmpfs             961096   169704    791392  18% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```
实验前查看一下硬盘使用情况。



##### 2.实验SQL插入【2千万行】
```
SCOTT@prod>set timing on
SCOTT@prod>create table bulk2 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:00.02
SCOTT@prod>insert into bulk2 nologging select * from scott.bulkinsert;
```



##### 3.实验结果
```
SCOTT@prod>insert into bulk2 nologging select * from scott.bulkinsert;

20000000 rows created.

Elapsed: 00:12:19.73
```


##### 4.运行脚本后：   
```
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME          VALUE
---------- ----------
redo size  1343777144
```


```
[oracle@bulkinsert ~]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 21821500   5286292  81% /
tmpfs             961096   479532    481564  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```



