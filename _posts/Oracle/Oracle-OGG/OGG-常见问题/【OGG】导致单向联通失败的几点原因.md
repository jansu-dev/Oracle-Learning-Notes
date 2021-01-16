---
title: 【OGG】导致单项连通OGG失败的几点原因  
date: 2019-10-31 
categories: Oracle
tags: Oracle-OGG
---



##### 三点原因：

- [在安装OGG的过程中，注意两库字符集要一致](在安装OGG的过程中，注意两库字符集要一致)
- [备库端，所要导入的表中不能有数据](备库端，所要导入的表中不能有数据)
- [在主库端ggsci中，对存在主外键依赖的多个表操作时，主键所在表应在外键所在表的前面](在主库端ggsci中，对存在主外键依赖的多个表操作时，主键所在表应在外键所在表的前面)



### 一、在安装OGG的过程中，注意两库字符集要一致  

##### 1.主库sys用户查表  
可以看到在scott用户模式下，在emp表中插入了中文字符。
```
SYS@prod>select * from scott.emp;

     EMPNO ENAME      JOB     MGR     HIREDATE            SAL   COMM   DEPTNO
---------- ------- ------   ------- -------------------   ----  ----   ------
      1234 张三                                                            10
      7369 SMITH     CLERK    7902   1980-12-17 00:00:00   800             20
      ......
      ......
      ......
      7902 FORD      ANALYST   7566   1981-12-03 00:00:00  3000            20
      7934 MILLER     CLERK    7782   1982-01-23 00:00:00  1300            10
      2222 c                                                               10
16 rows selected.
```

##### 2.主库ggsci中编写eini_1的extract捕获文件参数   

```
GGSCI (ogg1) 10> edit params eini_1

-- GoldenGate Initial Data Capture -- for EMP and EMP
--
EXTRACT EINI_1
SETENV (NLS_LANG=SIMPLIFIED CHINESE_CHINA.AL32UTF8)
USERID ogg, PASSWORD ogg
RMTHOST ogg2, MGRPORT 7809
RMTTASK REPLICAT, GROUP RINI_1
TABLE scott.dept;
TABLE scott.emp;  

:wq           
保存退出  



GGSCI (ogg1) 7> start eini_1

Sending START request to MANAGER ...
EXTRACT EINI_1 starting

```

##### 3.备库ggsci中编写rini_1的复制文件参数
```
GGSCI (ogg2) 2> edit params rini_1

-- GoldenGate Initial Load Delivery
--
REPLICAT RINI_1
SETENV (NLS_LANG=AMERICAN_AMERICA.ZHS16GBK)
ASSUMETARGETDEFS
USERID ogg, PASSWORD ogg
DISCARDFILE ./dirrpt/RINIaa.dsc, PURGE
MAP scott.emp, TARGET scott.emp;
MAP scott.dept, TARGET scott.dept;
```
可以看到捕获和复制两个参数文件中的SETENV (NLS_LANG=...)字符集不同。

##### 4.备库端查询结果   

```
SYS@prod>select * from scott.emp;

     EMPNO ENAME      JOB     MGR     HIREDATE            SAL   COMM   DEPTNO
---------- ------- ------   ------- -------------------   ----  ----   ------
      1234 寮犱笁                                                           10
      7369 SMITH     CLERK    7902   1980-12-17 00:00:00   800             20
      ......
      ......
      ......
      7902 FORD      ANALYST   7566   1981-12-03 00:00:00  3000            20
      7934 MILLER     CLERK    7782   1982-01-23 00:00:00  1300            10
      2222 c                                                               10
16 rows selected.
```
可以看到在导表过程中，出现了乱码问题。   

##### 5.解决办法
```
SYS@prod>select userenv('language') from dual;

USERENV('LANGUAGE')
----------------------------------------------------
SIMPLIFIED CHINESE_CHINA.AL32UTF8
```
在主备库分别使用上述语句查询数据库所用字符集，并改正主备库的参数文件（extract和replicat），使得两库字符集相同，已解决乱码问题。  





### 二、备库端，所要导入的表中不能有数据  
##### 1.备库端表中有数据会出现OGG-01203错误  
```
GGSCI (ogg1) 11> start eini_1

Sending START request to MANAGER ...
EXTRACT EINI_1 starting


GGSCI (ogg1) 12> view report eini_1


2019-10-30 11:17:49  INFO    OGG-01017  Wildcard resolution set t
o IMMEDIATE because SOURCEISTABLE is used.

*****************************************************************
******
                 Oracle GoldenGate Capture for Oracle
    Version 11.2.1.0.1 OGGCORE_11.2.1.0.1_PLATFORMS_120423.0230_F
BO
   Linux, x64, 64bit (optimized), Oracle 11g on Apr 23 2012 08:42
:16
 
Copyright (C) 1995, 2012, Oracle and/or its affiliates. All right
s reserved.


......
......
......


Database Language and Character Set:
NLS_LANG         = "SIMPLIFIED CHINESE_CHINA.AL32UTF8" 
NLS_LANGUAGE     = "AMERICAN" 
NLS_TERRITORY    = "AMERICA" 
NLS_CHARACTERSET = "AL32UTF8" 

...skipping one line
Processing table SCOTT.DEPT

2019-10-30 11:17:57  WARNING OGG-01194  EXTRACT task RINI_1 abend
ed : There is no trail to reposition to when doing direct load ta
sk.

Source Context :
  SourceModule            : [er.idlx]
  SourceID                : [/scratch/aime1/adestore/views/aime1_
adc4150256/oggcore/OpenSys/src/app/er/idlx.c]
......
......
......
ty_v0+0x38a) [0x4e8b7a]]

2019-10-30 11:17:57  ERROR   OGG-01203  EXTRACT abending.

2019-10-30 11:17:57  ERROR   OGG-01668  PROCESS ABENDING.
```

可以看到在view report eini_1的捕获报告文件中，报错OGG-01203。   


##### 2. 解决办法   

```
SYS@prod>truncate table scott.emp;

Table truncated.

SYS@prod>delete scott.dept;

4 rows deleted.

SYS@prod>commit;

Commit complete.

SYS@prod>select * from scott.emp;

no rows selected

SYS@prod>select * from scott.dept;

no rows selected
```
将备库端想要导入的表清空内容，再次在主库端的ggsci中执行start eini_1便可正常将表传输过来。  
注意：在备库端表中没有数据不代表可以删除表，传数据需要保留表结构，因为OGG实际上是在对备库端所要操作的表做insert操作。 





### 三、在主库端ggsci中，对存在主外键依赖的多个表操作时，主键所在表应在外键所在表的前面
##### 1.在主库端编写捕获进程参数文件
```
GGSCI (ogg1) 17> edit params eini_1

-- GoldenGate Initial Data Capture -- for EMP and EMP
--
EXTRACT EINI_1
SETENV (NLS_LANG=SIMPLIFIED CHINESE_CHINA.AL32UTF8)
USERID ogg, PASSWORD ogg
RMTHOST ogg2, MGRPORT 7809
RMTTASK REPLICAT, GROUP RINI_1
TABLE scott.emp;
TABLE scott.dept;
```
从如上代码中可以看出，dept表和和emp表之间存在主外键依赖，dept表为主键表，emp表为外键表。  
在如上代码中，我将外键表放在了前面。

##### 2.在主库端运行捕获进程eini_1
```
GGSCI (ogg1) 15> start eini_1

Sending START request to MANAGER ...
EXTRACT EINI_1 starting
```
##### 3.在备库端查询表中是否传入数据
```
SYS@prod>select * from scott.emp;

no rows selected

SYS@prod>select * from scott.dept;

no rows selected
```
发现在备库并未传入数据。


##### 4.主库端查看捕获进程报告
```
GGSCI (ogg1) 16> view report eini_1


2019-10-30 11:32:58  INFO    OGG-01017  Wildcard resolution set t
o IMMEDIATE because SOURCEISTABLE is used.

......
......
......

Database Language and Character Set:
NLS_LANG         = "SIMPLIFIED CHINESE_CHINA.AL32UTF8" 
NLS_LANGUAGE     = "AMERICAN" 
NLS_TERRITORY    = "AMERICA" 
NLS_CHARACTERSET = "AL32UTF8" 

...skipping one line
Processing table SCOTT.EMP

2019-10-30 11:33:04  WARNING OGG-01194  EXTRACT task RINI_1 abend
ed : There is no trail to reposition to when doing direct load ta
sk.

......
......
......

xfd) [0x3cc321ed1d]]
                          : [/u01/app/ogg/extract(__gxx_personali
ty_v0+0x38a) [0x4e8b7a]]

2019-10-30 11:33:04  ERROR   OGG-01203  EXTRACT abending.

2019-10-30 11:33:04  ERROR   OGG-01668  PROCESS ABENDING.
```
查看捕获进程报告，发现与备库端表中存在数据一样（详见：上方错误二）同样出现了OGG-01203错误。   
##### 5.解决办法   


```
GGSCI (ogg1) 18> edit params eini_1


-- GoldenGate Initial Data Capture -- for EMP and EMP
--
EXTRACT EINI_1
SETENV (NLS_LANG=SIMPLIFIED CHINESE_CHINA.AL32UTF8)
USERID ogg, PASSWORD ogg
RMTHOST ogg2, MGRPORT 7809
RMTTASK REPLICAT, GROUP RINI_1
TABLE scott.dept;
TABLE scott.emp;
~                 
```

将主键所在表写在外键所在表前面，问题便可解决，代码如上所示。




























