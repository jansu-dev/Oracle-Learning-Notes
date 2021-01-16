---
title: 防火墙与SELINUX使DG主备库失联导致备库开库失败
date: 2019-10-27  
categories: Oracle
tags: Oracle-DG
---




### 错误简介   
基本信息  ：  

- 主库：bj   
- 备库：tj

​       实验虚机双库不知什么原因主备库两库开库报错system文件闭不一致，故对主库进行recover。查询归档文件后，发现数据库要求使用214序列文件来对数据进行恢复。

### 在主库端：

```shell
[oracle@coresu ~]$ sqlplus / as sysdba

SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME    OPEN_MODE    DATABASE_ROLE    PROTECTION_MODE      SWITCHOVER_STATUS
---- -------------- -------------- --------------------- --------------------
PROD    MOUNTED       PRIMARY       MAXIMUM PERFORMANCE      NOT ALLOWED

SYS@prod>recover database ;
ORA-00283: 恢复会话因错误而取消 ORA-00264:
不要求恢复


SYS@prod>recover database using backup controlfile;
ORA-00279: 更改 2084032 (在 10/20/2019 16:58:44 生成) 对于线程 1 是必需的 ORA-00289:
建议: /u01/arch/1_214_1021841806.dbf
ORA-00280: 更改 2084032 (用于线程 1) 在序列 #214 中


SYS@prod>exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@coresu ~]$ cd /u01/arch/
[oracle@coresu arch]$ ll
total 34156
......
......
-rw-r-----. 1 oracle oinstall   130560 Oct 20 16:29 1_212_1021841806.dbf
-rw-r-----. 1 oracle oinstall     1024 Oct 20 16:34 1_213_1021841806.dbf
[oracle@coresu arch]$ sqlplus / as sysdba
SQL*Plus: Release 11.2.0.4.0 Production on 星期四 10月 24 15:19:48 2019
Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prod>recover database using backup controlfile until cancel;
ORA-00279: 更改 2084032 (在 10/20/2019 16:58:44 生成) 对于线程 1 是必需的 ORA-00289:
建议: /u01/arch/1_214_1021841806.dbf
ORA-00280: 更改 2084032 (用于线程 1) 在序列 #214 中


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
/u01/oradata/prod/redo02.log
Log applied.
Media recovery complete.
SYS@prod>alter database open;
alter database open
*
ERROR at line 1:
ORA-01589: 要打开数据库则必须使用 RESETLOGS 或 NORESETLOGS 选项


SYS@prod>alter database open resetlogs;

Database altered.

SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME	  OPEN_MODE	 DATABASE_ROLE	   PROTECTION_MODE      SWITCHOVER_STATUS
----- ------------ -------------- -------------------- --------------------
PROD	  READ WRITE	PRIMARY		     MAXIMUM PERFORMANCE   FAILED DESTINATION

```

### 在备库端：
```shell
[oracle@dg ~]$ lsnrctl status

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 24-10月-2019 15:32:18

Copyright (c) 1991, 2013, Oracle.  All rights reserved.

正在连接到 (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=dg)(PORT=1521)))
LISTENER 的 STATUS
------------------------
别名                      LISTENER
版本                      TNSLSNR for Linux: Version 11.2.0.4.0 - Production
启动日期                  24-10月-2019 14:36:44
正常运行时间              0 天 0 小时 55 分 33 秒
跟踪级别                  off
安全性                    ON: Local OS Authentication
SNMP                      OFF
监听程序参数文件          /u01/oracle/network/admin/listener.ora
监听程序日志文件          /u01/diag/tnslsnr/dg/listener/alert/log.xml
监听端点概要...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=dg)(PORT=1521)))
服务摘要..
服务 "tj" 包含 1 个实例。
  实例 "prodstd", 状态 READY, 包含此服务的 1 个处理程序...
命令执行成功

[oracle@dg ~]$ sqlplus / as sysdba
SQL*Plus: Release 11.2.0.4.0 Production on 星期四 10月 24 15:34:23 2019
Copyright (c) 1982, 2013, Oracle.  All rights reserved.

连接到: 
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prodstd>alter database open;
alter database open
*
第 1 行出现错误:
ORA-10458: standby database requires recovery
ORA-01196: 文件 1 由于介质恢复会话失败而不一致 ORA-01110:
数据文件 1: '/u01/oradata/prodstd/system01.dbf'

SYS@prodstd>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME      OPEN_MODE        DATABASE_ROLE    PROTECTION_MODE      SWITCHOVER_STATUS
--------- -------------------- ---------------- -------------------- --------------------
PROD      MOUNTED          PHYSICAL STANDBY MAXIMUM PERFORMANCE  NOT ALLOWED

```

##### 以上我对主库进行了不完全备份恢复使得主库，可以以resetlogs的方式正常开库。但是此时在主库查询v$database时，发现switchover_status为FAILED DESTINATION状态，尝试在备库尝试开库，结果报错ORA-01196: 文件 1 由于介质恢复会话失败而不一致。   
进行如下尝试：  
1. 最初以为是参数文件中配置的归档路径地址错误导致的，重配了参数文件的归档路径但仍未解决。    
2. 网上有说是备库端没有主库端的口令文件导致的，常识未果。（此种说发有待商榷，个人认为在DG中，备库无需主库的口令文件）   
3. 检查网络监听是否正常，未直接决绝但是提供了有效的信息。

### 在主库端：
```shell
SYS@prod>alter system register;

System altered.

SYS@prod>exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@coresu arch]$ lsnrctl status

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 24-10月-2019 15:23:31

Copyright (c) 1991, 2013, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=coresu)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 11.2.0.4.0 - Production
Start Date                24-10月-2019 14:46:23
Uptime                    0 days 0 hr. 37 min. 7 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/oracle/network/admin/listener.ora
Listener Log File         /u01/diag/tnslsnr/coresu/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=coresu)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "bj" has 1 instance(s).
  Instance "prod", status READY, has 1 handler(s) for this service...
Service "prodXDB" has 1 instance(s).
  Instance "prod", status READY, has 1 handler(s) for this service...
The command completed successfully

[oracle@coresu arch]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on 星期四 10月 24 15:23:34 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME	  OPEN_MODE	    DATABASE_ROLE	   PROTECTION_MODE      SWITCHOVER_STATUS
----- -------------- --------------- -------------------- --------------------
PROD	  READ WRITE	     PRIMARY	    MAXIMUM PERFORMANCE   FAILED DESTINATION

SYS@prod>archive log list
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       /u01/arch/
Oldest online log sequence     3
Next log sequence to archive   5
Current log sequence	       5

SYS@prod>select status from v$instance;

STATUS
------------
OPEN


SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME	  OPEN_MODE	    DATABASE_ROLE	  PROTECTION_MODE     SWITCHOVER_STATUS
------- ----------    -------------- -------------------  ------------------
PROD	  READ WRITE	     PRIMARY		 MAXIMUM PERFORMANCE  FAILED DESTINATION

```
##### 发现主库监听运行正常，并查询了主库相关信息。如：主库为归档模式、数据库处于开库状态且但是SWITCHOVER_STATUS还是处于FAILED DESTINATION状态，于是便测试了一下备库是否能连接主库。


### 在备库端：
```shell
[oracle@dg ~]$ sqlplus sys/oracle@bj as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on 星期四 10月 24 15:12:34 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

ERROR:
ORA-12543: TNS: 无法连接目标主机


请输入用户名:  exit
输入口令: 
ERROR:
ORA-01033: ORACLE initialization or shutdown in progress
进程 ID: 0
会话 ID: 0 序列号: 0


请输入用户名:  
ERROR:
ORA-01017: 用户名/口令无效; 登录被拒绝


SP2-0157: 在 3 次尝试之后无法连接到 ORACLE, 退出 SQL*Plus
```


##### 发现备库无法连接主库，说明主备库之间网络连接不通。猜测可能是由于网络连接出现问题导致主库归档日志无法传到备库，从而备库无法同步，导致备库的开库失败。在准备库查看是什么原因导致的网络连接不畅通，经检查两端LISTENER监听均属正常状态（可以从代码中看出）。最后恍然大悟是重启数据库后，临时关闭的selinux和iptables配置重新开启导致的网络连接失败。

### 在备库端：
```
[oracle@coresu arch]$ getenforce
Enforcing
[oracle@coresu arch]$ exit
logout
[root@coresu ~]# setenforce 0
[root@coresu ~]# getenforce
Permissive        
[root@coresu ~]# service iptables stop
iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
iptables: Flushing firewall rules:                         [  OK  ]
iptables: Unloading modules:                               [  OK  ]
[root@coresu ~]# chkconfig iptables off
```

### 在备库端：  
```shell  
[root@dg ~]# su - oracle
[oracle@dg ~]$ sqlplus sys/oracle@bj as sysdba
Copyright (c) 1982, 2013, Oracle.  All rights reserved.
连接到: 
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@bj>
SYS@bj>exit
[oracle@dg ~]$ sqlplus / as sysdba

SYS@prodstd>alter system register;

系统已更改。

SYS@prodstd>alter database open;

数据库已更改。

SYS@prodstd>recover managed standby database disconnect from session;
完成介质恢复。
SYS@prodstd>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME      OPEN_MODE       DATABASE_ROLE    PROTECTION_MODE   SWITCHOVER_STATUS
---- -------------------- --------------  ------------------ ------------------
PROD READ ONLY WITH APPLY PHYSICAL STANDBY MAXIMUM PERFORMANCE  NOT ALLOWED
```
##### 最后以"sqlplus sys/oracle@bj as sysdba"在备库连接主库正常。直接登录备库开库，正常打开。启动MRP进程应用归档日志进程，查询状态一切正常。至此问题解决。 
