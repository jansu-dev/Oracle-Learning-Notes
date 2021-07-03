---
title: [Oracle]-静默安装DBCA时ORA-04031问题解决
date: 2020-11-26
---



​		静默安装DBCA时,始终出现ORA-04031: unable to allocate 3981152 bytes of shared memory ("shared pool","unknown object","sga heap(1,0)","KCB Table Scan Buffer")问题，不断尝试后找到了解决办法。



### 环境简介

数据库版本：Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
Linux版本：Asianux Linux release 7.6.1810 (Core)
内核信息：Linux 3.10.0-957.axs7.x86_64 #1 SMP



### 问题现象

terminal敲入命令后；

```
静默安装命令： [oracle@XXXX ~]$ dbca  -silent  -createDatabase -responseFile   /soft/database/response/dbca.rsp
```

在日志中出现如下错误；

```
[oracle@XXXX ~]$ more /oracle/app/oracle/cfgtoollogs/dbca/XXXdb/XXXdb6.log
清除失败的步骤
DBCA_PROGRESS : 5%
复制数据库文件
DBCA_PROGRESS : 7%
ORA-04031: unable to allocate 3981152 bytes of shared memory ("shared pool","unknown
 object","sga heap(1,0)","KCB Table Scan Buffer")

DBCA_PROGRESS : 9%
DBCA_PROGRESS : 16%
DBCA_PROGRESS : 16%
ORA-01034: ORACLE not available

ORA-01034: ORACLE not available

DBCA_PROGRESS : 100%
```



### 解决方案：

从报错可以看出ORA-04031是一个与shared pool（内存）相关的的错误；
因为达不到shared pool的大小要求而无法创建数据库；
又因为在Oracle10g之后，sga采用sga_target参数自动管理内部各部分（shared pool、large pool...）大小；
故而，采用手动指定dbca的SGA与PGA总和的方式建库。

```
dbca  -silent  -createDatabase -totalMemory 102400  -responseFile   /soft/database/response/dbca.rsp
```

解决后日志打印结果：

```
[oracle@XXXX ~]$ tail -20 /oracle/app/oracle/cfgtoollogs/dbca/XXXdb/XXXdb7.log
DBCA_PROGRESS : 50%
DBCA_PROGRESS : 54%
DBCA_PROGRESS : 55%
DBCA_PROGRESS : 56%
DBCA_PROGRESS : 59%
DBCA_PROGRESS : 61%
将数据库注册到 Oracle Restart
DBCA_PROGRESS : 66%
正在进行数据库创建
DBCA_PROGRESS : 70%
DBCA_PROGRESS : 73%
DBCA_PROGRESS : 76%
DBCA_PROGRESS : 86%
DBCA_PROGRESS : 96%
DBCA_PROGRESS : 100%
数据库创建完成。有关详细信息, 请查看以下位置的日志文件:
 /oracle/app/oracle/cfgtoollogs/dbca/XXXdb。
数据库信息:
全局数据库名:XXXdb
系统标识符 (SID):XXXdb
```



### 参考文章：

- [参考文章1-dbca静默模式创建数据库](https://blog.csdn.net/martin201609/article/details/98260271)