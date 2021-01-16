---
title: DG中v$database的switchover_status字段sessions active阐述
date: 2019-10-26  
categories: Oracle
tags: Oracle-DG
---



### oracle官方原文：   
SESSIONS ACTIVE - The database has active sessions. On a physical standby database, the WITH SESSION SHUTDOWN SQL clause must be specified to perform a role transition while in this state. On a logical standby database, a role transition can be performed while in this state, but the role transition will not complete until all current transactions have committed.



### 作者翻译：

会话活跃态-当前数据库有活跃会话。在物理备库上出现会话活跃态时，必须使用with session shutdown子句来过度角色。在逻辑备库时，角色转换可以自动进行，但是所有事物如果没有被提交那么当前角色转换也不会完成。  



### 实验   

主库会话一：
```shell
coredeMacBook-Air:~ core$ ssh root@192.168.9.6
root@192.168.9.6 's password: 
Last login: Fri Oct 25 02:35:00 2019 from 192.168.9.102
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
[root@coresu ~]# su - oracle
[oracle@coresu ~]$ sqlplus / as sysdba
SQL*Plus: Release 11.2.0.4.0 Production on 星期五 10月 25 02:36:58 2019
Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME  OPEN_MODE   DATABASE_ROLE  PROTECTION_MODE      SWITCHOVER_STATUS
----  ----------  -------------  ------------------- -------------------
PROD  READ WRITE   PRIMARY       MAXIMUM PERFORMANCE  TO STANDBY

```

主库会话二：

```shell  
[oracle@coresu ~]$ ps -ef | grep LOCAL
oracle    7381  7380  0 02:36 ?        00:00:00 oracleprod (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))
oracle    7384  7302  0 02:37 pts/0    00:00:00 grep LOCAL
```


主库会话三：
```shell   
coredeMacBook-Air:~ core$ ssh root@192.168.9.6
root@192.168.9.6's password: 
Last login: Fri Oct 25 02:36:48 2019 from 192.168.9.102
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
[root@coresu ~]# su - oracle
[oracle@coresu ~]$ sqlplus  / as sysdba
SQL*Plus: Release 11.2.0.4.0 Production on 星期五 10月 25 02:39:34 2019
Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prod>

```


主库会话一：
```shell
coredeMacBook-Air:~ core$ ssh root@192.168.9.6
root@192.168.9.6 's password: 
Last login: Fri Oct 25 02:35:00 2019 from 192.168.9.102
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
[root@coresu ~]# su - oracle
[oracle@coresu ~]$ sqlplus / as sysdba
SQL*Plus: Release 11.2.0.4.0 Production on 星期五 10月 25 02:36:58 2019
Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME  OPEN_MODE   DATABASE_ROLE  PROTECTION_MODE      SWITCHOVER_STATUS
----  ----------  -------------  ------------------- -------------------
PROD  READ WRITE   PRIMARY       MAXIMUM PERFORMANCE  TO STANDBY

```

```shell
SYS@prod>select name,open_mode,database_role,protection_mode,switchover_status from v$database;

NAME   OPEN_MODE  DATABASE_ROLE   PROTECTION_MODE     SWITCHOVER_STATUS
----- ----------- -------------- ------------------- --------------------
PROD  READ WRITE  PRIMARY       MAXIMUM PERFORMANCE   SESSIONS ACTIVE

SYS@prod>
```


主库会话二：
```shell
[oracle@coresu ~]$ ps -ef | grep LOCAL
oracle    7381  7380  0 02:36 ?        00:00:00 oracleprod (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))
oracle    7446  7445  0 02:39 ?        00:00:00 oracleprod (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))
oracle    7449  7302  0 02:39 pts/0    00:00:00 grep LOCAL
[oracle@coresu ~]$ 
```





### 总结：  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由此可得，当会话连接超过一个时，switchover_status便会进入sessions active态。    
如果此时想切换主备库状态，必须使用with session shutdown子句以切断除此连接以外的其他session回话，才能将主库切换到备库态。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;有时，备库在进行recover managed standby database disconnect from session后，主库会出现一段时间的sessions active态，说明此时有其他终端与数据库建立了会话，去完成某写任务。


