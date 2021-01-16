---
title: 【OGG】解读DYNAMICRESOLUTION参数
date: 2019-11-3 
categories: Oracle
tags: Oracle-OGG
---



##### 官方描述：  

DYNAMICRESOLUTION | NODYNAMICRESOLUTION Valid for Extract and Replicat Use the DYNAMICRESOLUTION and NODYNAMICRESOLUTION parameters to control how table names are resolved. Use DYNAMICRESOLUTION to make processing start sooner when there is a large number of tables specified in TABLE or MAP statements. By default, whenever a process starts, Oracle GoldenGate queries the database for the attributes of the tables and then builds an object record for them. The record is maintained in memory and on disk, and the process of building it can be time-consuming if the database is large. DYNAMICRESOLUTION causes the object record to be built one table at a time, instead of all at once. A table’s attributes are added to the record the first time its object ID enters the transaction log, which occurs with the first extracted transaction on that table. Recordbuilding for other tables is deferred until activity occurs. DYNAMICRESOLUTION is the same as WILDCARDRESOLVE DYNAMIC. NODYNAMICRESOLUTION causes the object record to be built at startup. This option is not supported for Teradata. NODYNAMICRESOLUTION is the same as WILDCARDRESOLVE IMMEDIATE. For more information about WILDCARDRESOLVE, see page 389.  

##### coresu翻译：  
DYNAMICRESOLUTION | NODYNAMICRESOLUTION（动态解析|非动态解析）对捕获进程和和复制进程起作用，对于捕获进程和和复制进程使用两个参数去控制表何时被记录。  
当需要被确认的捕获表名或映射语句（MAP）特别大时，使用DYNAMICRESOLUTION参数能使进程快速启动。
缺省模式，每次启动进程，OGG都会访问数据库以获得表的属性，紧接着为这些表创建一个对象记录。  
这些记录被保存在内存和硬盘上，如果数据库很庞大，使用构建记录的过程会非常耗时。  
DYNAMICRESOLUTION动态解析，构建对象记录在第一次使用表时，而不是一次全部读入。 
事务一个日志在第一次事务被捕获时发生。其他表延迟创建对象记录直到活动发生时候。  
DYNAMICRESOLUTION与WILDCARDRESOLVE DYNAMIC功能一样。
NODYNAMICRESOLUTION构建记录在开启进程之前。此选项不对Teradata提供支持。
NODYNAMICRESOLUTION与WILDCARDRESOLVE IMMEDIATE功能一样。

##### 参考文档

[参考博文1--OGG官方文档](https://www.oracle.com/technetwork/database/availability/8398-goldengate-integrated-capture-1888658.pdf)

[参考博文2--Focus on Oracle博文](https://www.2cto.com/database/201409/334739.html)





