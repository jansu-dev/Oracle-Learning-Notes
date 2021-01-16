---
title: alter database open read only 为何还要提示recovery的原因
date: 2019-10-11
categories: Oracle
tags: Oracle常见问题
---


```shell
SQL> shutdown abort
ORACLE 例程已经关闭。
SQL> startup mount;
ORACLE 例程已经启动。

Total System Global Area  171966464 bytes
Fixed Size                   787988 bytes
Variable Size             145750508 bytes
Database Buffers           25165824 bytes
Redo Buffers                 262144 bytes
数据库装载完毕。
SQL> recover database;
完成介质恢复。
SQL> alter database open read only;
alter database open read only
*
第 1 行出现错误:
ORA-16005: 数据库需要恢复
为何还提示恢复呢？
```

##### 测试一：

```shell
SQL> shutdown abort
ORACLE 例程已经关闭。
SQL> startup mount;
ORACLE 例程已经启动。

Total System Global Area  171966464 bytes
Fixed Size                   787988 bytes
Variable Size             145750508 bytes
Database Buffers           25165824 bytes
Redo Buffers                 262144 bytes
数据库装载完毕。
SQL> recover database;
完成介质恢复。


SQL> select checkpoint_change# from v$database;

CHECKPOINT_CHANGE#
------------------
            749283

SQL> select nvl(last_change#,0),checkpoint_change# from v$datafile;

NVL(LAST_CHANGE#,0) CHECKPOINT_CHANGE#
------------------- ------------------
             749454             749454
             749454             749454
             749454             749454
             749454             749454
             749454             749454

SQL> select checkpoint_change# from v$datafile_header;

CHECKPOINT_CHANGE#
------------------
            749454
            749454
            749454
            749454
            749454
```

##### 测试二：

```shell
SQL> shutdown immediate
数据库已经关闭。
已经卸载数据库。
ORACLE 例程已经关闭。
SQL> startup mount;
ORACLE 例程已经启动。

Total System Global Area  171966464 bytes
Fixed Size                   787988 bytes
Variable Size             145750508 bytes
Database Buffers           25165824 bytes
Redo Buffers                 262144 bytes
数据库装载完毕。
SQL> select checkpoint_change# from v$datafile_header;

CHECKPOINT_CHANGE#
------------------
            769749
            769749
            769749
            769749
            769749

SQL> select nvl(last_change#,0),checkpoint_change# from v$datafile;

NVL(LAST_CHANGE#,0) CHECKPOINT_CHANGE#
------------------- ------------------
             769749             769749
             769749             769749
             769749             769749
             769749             769749
             769749             769749

SQL> select checkpoint_change# from v$database;

CHECKPOINT_CHANGE#
------------------
            769749

SQL> alter database open read only;

数据库已更改。
```
### 原因阐述：
​		可以看到系统检查点和数据文件检查点不一致，虽然recover database 把datafile的last_change#由null恢复成了checkpoint_change#，但 open read only 是会立即系统scn，datafile scn，datafile header scn冻结，由于本身三者就不一致，所以不能open read only。
