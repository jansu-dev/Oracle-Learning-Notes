---
title:[Oracle]--DG基于时间点增量恢复
date:2020-08-10
---



DG基于时间点增量恢复的案例。

- [环境制造](环境制造)
- [恢复操作](恢复操作)
- [检查操作](检查操作)



### 环境制造

可以使用各种方法制造缺归档GAP，
我是使用的是关网卡、关监听、切归档删归档的办法。



### 恢复操作

备库在有GAP的情况下，备库查看MRP进程期待哪个归档。

```
SQL> select process,status,sequence#,blocks from gv$managed_standby;

PROCESS   STATUS    SEQUENCE#     BLOCKS
--------- ------------ ---------- ----------
ARCH      CONNECTED        0       0
ARCH      CONNECTED        0       0
ARCH      CONNECTED        0       0
ARCH      CONNECTED        0       0
RFS      IDLE            0       0
RFS      IDLE                  289       1
RFS      IDLE            0       0
RFS      IDLE            0       0
MRP0      WAIT_FOR_GAP              228       0

9 rows selected.
```

主库依据归档SEQUENCE查出当时的scn--4692613

```
SQL> col name for a30
SQL> set linesize 120
SQL> select NAME,THREAD#,COMPLETION_TIME ,sequence#,first_change# from v$archived_log  where sequence#=228;

NAME                  THREAD# COMPLETION_TIME      SEQUENCE# FIRST_CHANGE#
------------------------------ ---------- ------------------- ---------- -------------
/u01/arch/1_228_1041664860.dbf    1 2020-06-19 08:52:12         228       4692613
uni_dg2             1 2020-06-19 08:52:40         228       4692613
```

对主库依据scn进行增量备份。

```
RMAN> backup as backupset INCREMENTAL from scn 4692613 database format '/u01/myrman/for_std_%U';

Starting backup at 2020-06-20 03:55:18
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
..................
..................
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2020-06-20 03:57:25
```

在主库为备库备份控制文件

```
RMAN> backup current controlfile for standby format '/u01/myrman/standby_ctl.bak';

Starting backup at 2020-06-20 04:07:51
.............
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2020-06-20 04:07:54
```

传递备份文件到备库。

```
[oracle@dg1 myrman]$ scp * oracle@192.168.169.201:/u01/myrman/
oracle@192.168.169.201's password: 
for_std_3sv36n16_1_1                                                                                    100%  258MB 129.2MB/s   00:02    
for_std_3tv36n53_1_1                                                                                    100%   10MB  10.2MB/s   00:01    
standby_ctl.bak                                                                                         100%   10MB  10.2MB/s   00:01    
```

在备库进行rman备库恢复。

```
rman target /

RMAN> catalog start with '/u01/myrman/';

using target database control file instead of recovery catalog
searching for all files that match the pattern /u01/myrman/

List of Files Unknown to the Database
=====================================
File Name: /u01/myrman/standby_ctl.bak
File Name: /u01/myrman/for_std_3sv36n16_1_1
File Name: /u01/myrman/for_std_3tv36n53_1_1

Do you really want to catalog the above files (enter YES or NO)? yes


RMAN> shutdown immediate;

RMAN> startup nomount;

RMAN> restore standby controlfile from '/u01/myrman/standby_ctl.bak';

Starting restore at 2020-06-20 04:14:47
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=134 device type=DISK

channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
output file name=/u01/oradata/prodstd/control01.ctl
output file name=/u01/fast_recovery_area/prodstd/control02.ctl
Finished restore at 2020-06-20 04:14:49


RMAN> alter database mount;

RMAN> recover database noredo; 

Starting recover at 2020-06-20 04:16:09
Starting implicit crosscheck backup at 2020-06-20 04:16:09
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=14 device type=DISK
Crosschecked 102 objects
Finished implicit crosscheck backup at 2020-06-20 04:16:10

Starting implicit crosscheck copy at 2020-06-20 04:16:10
using channel ORA_DISK_1
Crosschecked 2 objects
Finished implicit crosscheck copy at 2020-06-20 04:16:10

searching for all files in the recovery area
cataloging files...
no files cataloged

using channel ORA_DISK_1

Finished recover at 2020-06-20 04:16:11
```

为实时应用，增加备库日志组。
如果出现ORA-27038错误，需进入数据文件目录手动删除该日志文件。

```
SQL> alter database add standby logfile group 4 ('/u01/oradata/prodstd/redo04.log') size 50m;
alter database add standby logfile group 4 ('/u01/oradata/prodstd/redo04.log') size 50m
*
ERROR at line 1:
ORA-00301: 添加日志文件 '/u01/oradata/prodstd/redo04.log' 时出错 - 无法创建文件 ORA-27038:
所创建的文件已存在
Additional information: 1


[oracle@dg2 myrman]$ cd /u01/oradata/prodstd

[oracle@dg2 prodstd]$ ll |grep redo
-rw-r----- 1 oracle oinstall   52429312 Jun 19 23:48 redo01.log
-rw-r----- 1 oracle oinstall   52429312 Jun 19 23:48 redo02.log
-rw-r----- 1 oracle oinstall   52429312 Jun 19 23:49 redo03.log
-rw-r----- 1 oracle oinstall   52429312 Jun 20 03:10 redo04.log
-rw-r----- 1 oracle oinstall   52429312 Jun 20 01:16 redo05.log
-rw-r----- 1 oracle oinstall   52429312 Jun 19 23:44 redo06.log


[oracle@dg2 prodstd]$ rm -rf redo04.log 
[oracle@dg2 prodstd]$ rm -rf redo05.log 
[oracle@dg2 prodstd]$ rm -rf redo06.log 


SQL> alter database add standby logfile group 4 ('/u01/oradata/prodstd/redo04.log') size 50m;

Database altered.

SQL> alter database add standby logfile group 5 ('/u01/oradata/prodstd/redo05.log') size 50m;

Database altered.

SQL> alter database add standby logfile group 6 ('/u01/oradata/prodstd/redo06.log') size 50m;

Database altered.

SQL>  alter database add standby logfile group 7 ('/u01/oradata/prodstd/redo07.log') size 50m;

Database altered.


SQL> select group#,thread#,sequence#,bytes/1024/1024 m,blocksize,status from v$standby_log;

    GROUP#    THREAD#  SEQUENCE#      M  BLOCKSIZE STATUS
---------- ---------- ---------- ---------- ---------- ----------
     4        0           0     50       512 UNASSIGNED
     5        0           0     50       512 UNASSIGNED
     6        0           0     50       512 UNASSIGNED
     7        0           0     50       512 UNASSIGNED
```

启动MRP进程，应用归档。

```
SQL> alter database recover managed standby database using current logfile disconnect from session;

Database altered.
```



### 检查操作

检查恢复后控制文件和数据文件的scn是否一致。

```
SQL> select checkpoint_change#,checkpoint_time from v$datafile_header;

CHECKPOINT_CHANGE# CHECKPOINT_TIME
------------------ -------------------
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18

6 rows selected.

SQL> select checkpoint_change#,checkpoint_time from v$datafile;

CHECKPOINT_CHANGE# CHECKPOINT_TIME
------------------ -------------------
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18
       4791388 2020-06-20 03:55:18

6 rows selected.
```

检查备库检查日志组是否正常。

```
SQL> select group#,thread#,BYTES/1024/1024/1024 logsize,MEMBERS, status from v$log;

    GROUP#    THREAD#     LOGSIZE    MEMBERS STATUS
---------- ---------- ---------- ---------- ----------------
     1        1 .048828125      1 CLEARING
     3        1 .048828125      1 CURRENT
     2        1 .048828125      1 CLEARING



SQL> select group#,thread#,sequence#,bytes/1024/1024 m,blocksize,status from v$standby_log;

no rows selected
```

打印alert 日志信息。

```
alter database recover managed standby database using current logfile disconnect from session
Attempt to start background Managed Standby Recovery process (prodstd)
Sat Jun 20 04:44:02 2020
MRP0 started with pid=20, OS id=71429 
MRP0: Background Managed Standby Recovery process started (prodstd)
 started logmerger process
Sat Jun 20 04:44:07 2020
Managed Standby Recovery starting Real Time Apply
Parallel Media Recovery started with 2 slaves
Waiting for all non-current ORLs to be archived...
All non-current ORLs have been archived.
Media Recovery Log /u01/arch/1_289_1041664860.dbf
Completed: alter database recover managed standby database using current logfile disconnect from session
Media Recovery Log /u01/arch/1_290_1041664860.dbf
Media Recovery Waiting for thread 1 sequence 291 (in transit)
```

在备库查看归档是否应用。

```
SQL> col name for a30
SQL> SELECT SEQUENCE#,DEST_ID,NAME,APPLIED FROM V$ARCHIVED_LOG ORDER BY SEQUENCE#;

 SEQUENCE#    DEST_ID NAME                 APPLIED
---------- ---------- ------------------------------ ---------
       289        2 /u01/arch/1_289_1041664860.dbf YES
       290        2 /u01/arch/1_290_1041664860.dbf YES
```

在主库查看归档是否应用。

```
SQL> col name for a30
SQL> SELECT SEQUENCE#,DEST_ID,NAME,APPLIED FROM V$ARCHIVED_LOG ORDER BY SEQUENCE#;

 SEQUENCE#    DEST_ID NAME                 APPLIED
---------- ---------- ------------------------------ ---------
       287        2 uni_dg2                 YES
       288        1 /u01/arch/1_288_1041664860.dbf NO
       288        2 uni_dg2                 YES
       289        1 /u01/arch/1_289_1041664860.dbf NO
       289        2 uni_dg2                 YES
       289        2 uni_dg2                 YES
       290        1 /u01/arch/1_290_1041664860.dbf NO
       290        2 uni_dg2                 YES

316 rows selected.
```

备库查看状态，如果发现之前处于mount态启动的MRP，
需开库重新启动MRP进程，否则无法使备库处于open状态。

```
SQL> set linesize 120
col DB_UNIQUE_NAME for a10
select NAME,OPEN_MODE,SWITCHOVER_STATUS,DB_UNIQUE_NAME,CURRENT_SCN from v$database;SQL> SQL> 

NAME                OPEN_MODE      SWITCHOVER_STATUS     DB_UNIQUE_ CURRENT_SCN
------------------------------ -------------------- -------------------- ---------- -----------
PROD                   MOUNTED    NOT ALLOWED      uni_dg2    4793099

SQL> alter database recover managed standby database cancel;

Database altered.

SQL>  alter database open read only;

Database altered.

SQL> alter database recover managed standby database using current logfile disconnect from session;

Database altered.

SQL> select NAME,OPEN_MODE,SWITCHOVER_STATUS,DB_UNIQUE_NAME,CURRENT_SCN from v$database;

NAME                   OPEN_MODE        SWITCHOVER_STATUS     DB_UNIQUE_ CURRENT_SCN
------------------------------ -------------------- -------------------- ---------- -----------
PROD                   READ ONLY WITH APPLY NOT ALLOWED      uni_dg2    4793099
```

##### 至此，增量恢复结束!!!