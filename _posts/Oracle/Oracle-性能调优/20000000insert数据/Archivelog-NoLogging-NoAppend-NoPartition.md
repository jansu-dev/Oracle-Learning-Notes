---
title: 批量插入操作实验  之【Archivelog-NoLogging-NoAppend-NoPartition】
date: 2019-11-03
---




##### 1.实验运行脚本前：
```
SYS@prod>archive log list;
Database log mode          Archive Mode
Automatic archival         Enabled
Archive destination        /u01/arch
Oldest online log sequence     13378
Next log sequence to archive   13380
Current log sequence           13380
```
数据库为归档模式

```
SYS@prod>col name for a10;
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME        VALUE
---------- ----------
redo size   86252
```
在实验前查询归档日志大小

```
[oracle@bulkinsert arch]$ ll
total 0
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 24054084   3053708  89% /
tmpfs             961096   175320    785776  19% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```
查看磁盘使用情况。



##### 2.实验SQL插入【2千万行】
```shell
SYS@prod>create table scott.bulk5 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:02.34
SYS@prod>conn scott/scott
Connected.
SCOTT@prod>insert into bulk5 nologging select * from scott.bulkinsert;
```



##### 3.实验结果
```
SCOTT@prod>insert into bulk5 nologging select * from scott.bulkinsert;

20000000 rows created.

Elapsed: 00:13:50.00
```


##### 4.运行脚本后：   
```
SYS@prod>select name ,value from v$sysstat where name='redo size';

NAME         VALUE
---------- ----------
redo size  1342378436
```


```
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 26656068    451724  99% /
tmpfs             961096   477524    483572  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
[oracle@bulkinsert arch]$ ll
total 1352140
-rw-r-----. 1 oracle oinstall 5241344 Oct 25 19:42 1_13380_1022509032.log
-rw-r-----. 1 oracle oinstall 5107200 Oct 25 19:42 1_13381_1022509032.log
......
......
[oracle@bulkinsert arch]$ rm *
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25303928   1803864  94% /
tmpfs             961096   477524    483572  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```



