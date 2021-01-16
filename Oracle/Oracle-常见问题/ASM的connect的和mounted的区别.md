---
title: ASM的connect的和mounted的区别
date: 2019-10-08
categories: Oracle
tags: Oracle-RAC
---



## 解释问题：为什么查看v$asm_diskgroup的state有时是mounted有时connected？

### 实验：  

#### 	前提条件：
​		节点一：node1   
​		节点二：node2  


##### 注意：此时的用户为Oracle用户
```shell
[root@core1 ~]# su - oracle
[oracle@core1 ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Fri Oct 18 15:29:21 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Data Mining and Real Application Testing options

SQL> select name,state from v$asm_diskgroup;

NAME                   STATE
--------------------- -----------
DATA                   CONNECTED
FRA                    CONNECTED
OCR_VOTE               MOUNTED
```

##### 注意：此时的用户为Oracle用户
```shell
[root@core2 ~]# su - oracle
[oracle@core2 ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Thu Oct 17 02:30:43 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Data Mining and Real Application Testing options

SQL> show parameter name

NAME                     TYPE    VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name           string
db_file_name_convert             string
db_name                  string  prod
db_unique_name               string  prod
global_names                 boolean     FALSE
instance_name                string  prod2
lock_name_space              string
log_file_name_convert            string
processor_group_name             string
service_names                string  prod
```

##### 注意：此时的用户为grid用户
```shell  
[grid@core1 ~]# su - oracle
[grid@core2 ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Thu Oct 17 01:48:22 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Real Application Clusters and Automatic Storage Management options

SQL> select name,state from v$asm_diskgroup;

NAME                   STATE
--------------------- -----------
DATA                   MOUNTED
FRA                    MOUNTED
OCR_VOTE               MOUNTED
```


##### 注意：此时的用户为grid用户
```shell
[root@core2 ~]# su - grid
[grid@core2 ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Thu Oct 17 02:29:57 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Real Application Clusters and Automatic Storage Management options

SQL> show parameter name

NAME                     TYPE    VALUE
------------------------------------ ----------- ------------------------------
db_unique_name               string  +ASM
instance_name                string  +ASM2
lock_name_space              string
service_names                string  +ASM

```


### 总结：  
[官方文档说明](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-ASM_DISKGROUP.html#GUID-5CF77719-75BE-4312-84A3-49A7C6A20393)  

connected状态:是指磁盘组正在被数据库实例使用，不论该数据库当前是否处于关机状态。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表现：在oracle用户下，查询某些磁盘组状态为connected
mounted状态:是指磁盘组成功地准备服务为对应的数据库，即表示挂载成功以备使用。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表现：在grid用户下，查询磁盘组状态为mounted,因为数据库没有grid用户  
使用sqlplus / as sysdba登录数据库的时候,登录的是ASM的实例从show parameter name命令还是那个便可看出来。两个磁盘组是被Oracle所使用，而不是被ASM磁盘组所使用。    

故至此，此问题以完美解释！  