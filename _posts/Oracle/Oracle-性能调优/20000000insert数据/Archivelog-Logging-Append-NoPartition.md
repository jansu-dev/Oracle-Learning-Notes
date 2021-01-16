---
title: 批量插入操作实验  之【Archivelog-Logging-Append-NoPartition】
date: 2019-11-02
---




##### 1.实验运行脚本前：
```
SYS@prod>archive log list
Database log mode        Archive Mode
Automatic archival         Enabled
Archive destination        /u01/arch
Oldest online log sequence     13942
Next log sequence to archive   13944
Current log sequence           13944
```
数据库为归档模式。

```
SYS@prod>col name for a10;
SYS@prod>select name,value from v$sysstat where name='redo size';

NAME           VALUE
---------- ----------
redo size      529088

```
在实验前查询归档日志大小。

```
[oracle@bulkinsert arch]$ ll
total 0
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25305052   1802740  94% /
tmpfs             961096   232016    729080  25% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```

查看一下磁盘空间是使用情况。


##### 2.实验SQL插入【2千万行】
```
SCOTT@prod>create table bulk7 as select * from scott.bulkinsert where 1=2;

Table created.

Elapsed: 00:00:02.44
SCOTT@prod>insert /*+ append */ into bulk7 select * from bulkinsert;

```

##### 3.实验结果
```
SCOTT@prod>insert /*+ append */ into bulk7 select * from bulkinsert;

20000000 rows created.

Elapsed: 00:07:24.94
```


##### 4.运行脚本后：   
```
SYS@prod>select name,value from v$sysstat where name='redo size';

NAME    VALUE
---------- ----------
redo size  1320251776
```


```
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 26632284    475508  99% /
tmpfs             961096   474396    486700  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
[oracle@bulkinsert arch]$ ll
total 1326800
-rw-r-----. 1 oracle oinstall 5238272 Oct 25 21:02 1_13944_1022509032.log
-rw-r-----. 1 oracle oinstall 5101568 Oct 25 21:02 1_13945_1022509032.log
......
......
[oracle@bulkinsert arch]$ rm *
[oracle@bulkinsert arch]$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda3       28565504 25305484   1802308  94% /
tmpfs             961096   474396    486700  50% /dev/shm
/dev/sda1         194241    36391    147610  20% /boot
/dev/sr0         3795078  3795078         0 100% /media
```



