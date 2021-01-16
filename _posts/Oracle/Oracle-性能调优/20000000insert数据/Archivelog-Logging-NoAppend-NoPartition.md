---
title: 批量插入操作实验  之【Archivelog-Logging-NoAppend-NoPartition】
date: 2019-11-02
---




##### 1.实验运行脚本前：
```
SYS@prod>archive log list
Database log mode        Archive Mode
Automatic archival         Enabled
Archive destination        /u01/arch
Oldest online log sequence     13649
Next log sequence to archive   13651
Current log sequence         13651
```
数据库为归档模式

```
SYS@prod>col name for a10;
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME          VALUE
---------- ----------
redo size     218692
```
在实验前查询归档日志大小

```
[oracle@bulkinsert arch]$ ll
total 0
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25304056   1803736  94% /
tmpfs             961096   233444    727652  25% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```
实验前查看一下硬盘使用情况。



##### 2.实验SQL插入【2千万行】
```
SCOTT@prod>create table bulk6 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:01.95
SCOTT@prod>insert into bulk6 select * from bulkinsert;
```



##### 3.实验结果
```
SCOTT@prod>insert into bulk6 select * from bulkinsert;

20000000 rows created.

Elapsed: 00:13:51.66
```


##### 4.运行脚本后：   
```
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME    VALUE
---------- ----------
redo size  1342060868
```


```
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 26656776    451016  99% /
tmpfs             961096   477304    483792  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
[oracle@bulkinsert arch]$ ll
total 1352136
-rw-r-----. 1 oracle oinstall 5237248 Oct 25 20:08 1_13651_1022509032.log
-rw-r-----. 1 oracle oinstall 5101568 Oct 25 20:08 1_13652_1022509032.log
...... 
......
[oracle@bulkinsert arch]$ rm *
[oracle@bulkinsert arch]$ ll
total 0
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25304640   1803152  94% /
tmpfs             961096   477304    483792  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```



