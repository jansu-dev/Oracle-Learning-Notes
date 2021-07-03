---
title:[OGG]-11.2.0.4版本Oracle ASM实例Extract进程ORA-26947问题解决
date:2020-11-09
---



11.2.0.2 ~ 11.2.0.5版本的Oracle采用ASM存储出现extract无法读取归档情况的问题解决。

- [问题描述](问题描述)
- [解决方法](解决方法)
- [参考文章](参考文章)



### 问题描述

- **无法读取ASM磁盘中归档**

因为搭建的是单实例ASM版本，报错如下，经查找发现是因为asm版本OGG在11.2.0.2 ~ 11.2.0.5版本的Oracle采用ASM存储时，需要在抽取进程中配置TRANLOGOPTIONS DBLOGREADER参数才可以。

```
2020-11-09 09:06:15  ERROR   OGG-00446  Opening ASM file +DATA/prod/onlinelog/group_1.261.859325521 in DBLOGREADER mode: (26947) ORA-26947: Oracle GoldenGate
Not able to establish initial position for begin time 2014-09-28 14:31:42.

2020-11-09 09:06:15  ERROR   OGG-01668  PROCESS ABENDING.
```

- **ogg用户权限不足**

本次实验时，在解决无法读取ASM磁盘中归档日志后，又出现了新权限不足的问题，最后通过授予DBA最高权限角色解决。

```
2020-11-09 09:06:29  ERROR   OGG-00446  Opening file +DATA1/prod/onlinelog/gr
oup_1.257.1055743039 in DBLOGREADER mode: (1031) ORA-01031: insufficient priv
ileges
Not able to establish initial position for begin time 2020-11-09 09:03:45.

2020-11-09 09:06:29  ERROR   OGG-01668  PROCESS ABENDING.
```



### 解决方法

- **ASM归档问题解决**

```
GGSCI (jandb) 1> view params E_TEST

EXTRACT E_TEST
SETENV (NLS_LANG=AMERICAN_AMERICA.ZHS16GBK)
TRANLOGOPTIONS  DBLOGREADER
USERID ogg, PASSWORD ogg
EXTTRAIL /lsi/ggs/dirdat/t1
TABLE jan.test;
```

追加 TRANLOGOPTIONS DBLOGREADER参数后，重启extract进程。

- **权限问题解决**

```
SQL > grant dba to ogg;
```

- **参考ID 1061093.1**

参考：Extract fail due to an ASM connection configuration issue [ID 1061093.1]

Applies to:
Oracle GoldenGate - Version 11.1.1.0.0 and later
Information in this document applies to any platform.
Goal
To show how to recover from an extract failure when your Archive or Redo files are stored under ASM
and you see one of the following messages

ERROR 118 No Valid Log File For Current Redo Sequence Xxxx, Thread Y

ERROR 500 No valid log files for current redo sequence X, thread Y, error retrieving redo file name for sequence X, archived = 0, use_alternate = 0 Not able to establish initial position for begin time YYYY-MM-DD HH:MI:SS

ERROR OGG-00446 error 2 (No such file or directory) opening redo log .dbf for sequence ####
Not able to establish initial position for begin time YYYY-MM-DD HH:MI:SS
Fix
If you are running Oracle ASM, the problem may be that the ASM connection is either not defined or is incorrectly defined or TRANSLOGOPTINS DBLOGREADER needs to be added. If your archive files are ONLY under ASM and extract receives an error 500, extract may have run successfully until the process needed to read from the ARCHIVES instead of the REDO. Once it needs to read from archive, the extract will fail.

Please Add the following line, or correct it in your Extract parameter file, if you are On Oracle 11.2.0.2 or better, or 10.2.0.5 or better and using OGG 11.x
TRANLOGOPTIONS DBLOGREADER
If the above version of Oracle or OGG doesn't apply to you specifying a user that can connect to the ASM instance and restart your Extract:

TRANLOGOPTIONS ASMUSER @,
ASMPASSWORD



### 参考文章

- [参考文章1- itpub-snowdba的文章](http://blog.itpub.net/29047826/viewspace-1283722/)
- [参考文章2-csdn-Demonson的文章](https://blog.csdn.net/demonson/article/details/79473181)