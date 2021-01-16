---
title: 【调优】--如何优化批量插入数据（导表）
date: 2019:11:8
categories: Oracle
tags: Oracle调优
---



## 如何优化对一个表在短时间内进行大批量的insert操作



### 一、一般解决办法  

* 解除索引限制 index  
* 不记录日志 nologging

面对大批量操着时，首先想到由于短时间被产生大量的DML操作，导致Redo Log大量产生，想到一般解决办法，即如何合理控制日志的产生。     

##### 1. 优先考虑是否能去掉索引  
* 【原因】： 因为索引对insert的影响很大，在数据库中的所有的插入操作都将被记录在日志中，随着操作的增加日志数量的暴增，LGWR后台进程在将日志数据从Redo Log Buffer以串行的方式刷入Redo Log File中出现拥挤现象，从而造成批量插入数据速度慢。   
* 【方法】：在批量导表时，可以考虑现将索引去掉，等待所有数据出入完成之后再重建索引。 
* 【经验】：千万级的数据的批量插入操作产生归档arch非常快，需在操作系统层面使用df -h命令关注归档的产生量，  
及时使用RMAN或者UMAN进行备份，避免arch目录撑爆导致的数据量丢失现象。

##### 2. nologging不记录日志  
* 【原因】：同上
* 【方法】：在上面我们已经分析了，导致批量插入数据慢的原因之一可能是，日志拥挤导致的。因此，可以单独对某个表采用不记录日志策略。nologging和append可以配合使用，append表示在表的高水位线以上进行插入操作，不去访问位图或者free list表，因此便大大加快了插入的速度。
* 【注意】：append会对表加排它锁，阻塞除select意外的所有DML操作。
* 【注意】：这里的不记录日志仅是尽量少的记录日志，并不是一点都不记录。

##### 单独使某个表不记录日志代码

```
alter table [table_name] nologging;    
insert /*+ append */ into tab1 select * from tab2;    
commit;   
alter table tab1 logging;  
```

##### 【表格】--oracle日志三模式：  
| 模式 | 概述 | 限制级别 |
| :--- | :--- | :--- |
| logging | 记录日志（缺省） | 表级 |
| nologging | 尽力减少记录日志 | 表级 |
| force logging | 强制记录日志  | 数据库级或表级别 |

在本次实验中，为减小日志产生，所有试验的force logging模式均为关闭状态。   
```
SYS@prod>select force_logging from v$database;

FOR
---
NO
```

##### 【表格】--经过博主大量实验append、logging、归档模式组合得到如下参数表：

| archivelog | logging | append | 所用时间（分:秒） | 所用时间（秒） | 增加Redo（Bytes） | 备注 |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 是 | 否 | 否 | 13:50.00 | 830 | 1342292184 | 生产环境 |
| 是 | 是 | 否 | 13:51.66 | 831.66 | 1341842176 | 生产环境 |
| 是 | 否 | 是 | 06:51.72 | 411.72 | 1318987152 | 生产环境 |
| 是 | 是 | 是 | 07:24.94 | 444.94 | 1319722688 | 生产环境 |
| 否 | 否 | 是 | 01:14.60 | 74.6 | 848832 | 非生产环境 |
| 否 | 是 | 是 | 01:36.28 | 96.28 | 853476 | 非生产环境 |
| 否 | 否 | 否 | 12:19.73 | 739.73 | 1343727432 | 非生产环境 |
| 否 | 是 | 否 | 12:37.08 | 757.08 | 1342402072 | 非生产环境 |

#### 不同归档模式下组合耗时统计图示   
![耗时统计图示](http://cdn.lifemini.cn/dbblog/20210115/e9cbeb80f48a404b88126f6f33d63b19.png)

#### 具体实验数据及过程   
* [Archivelog-Logging-Append-NoPartition](./20000000insert数据/Archivelog-Logging-Append-NoPartition.md)
* [Archivelog-Logging-NoAppend-NoPartition](./20000000insert数据/Archivelog-Logging-NoAppend-NoPartition.md)
* [Archivelog-NoLogging-Append-NoPartition](./20000000insert数据/Archivelog-NoLogging-Append-NoPartition.md)
* [Archivelog-NoLogging-NoAppend-NoPartition](./20000000insert数据/Archivelog-NoLogging-NoAppend-NoPartition.md)
* [NoArchivelog-Logging-Append-NoPartition](./20000000insert数据/NoArchivelog-Logging-Append-NoPartition.md)
* [NoArchivelog-Logging-NoAppend-NoPartition](./20000000insert数据/NoArchivelog-Logging-NoAppend-NoPartition.md)
* [NoArchivelog-NoLogging-Append-NoPartition](./20000000insert数据/NoArchivelog-NoLogging-Append-NoPartition.md)
* [NoArchivelog-NoLogging-NoAppend-NoPartition](./20000000insert数据/NoArchivelog-NoLogging-NoAppend-NoPartition.md)


#### 从耗时图中可以看出

* 非归档模式下，使用append减少redo日志的产生，在实验中数据反映最为明显。  
* 归档模式下，使用append+logging或者使用append+nologging对redo日志的产生量来说差不多，均对日志有一定优化。
* 【注意】非归档条件下的解决办法有个致命的缺点就是需要关机到mount状态，改为noarchivelog状态。虽然这样的效果很显著，但是在生产库上往往是不允许的。  




### 二、特殊解决办法：  

* 使用批量绑定&#32;bulk&#32;binding
* 并行插入方法&#32;parallel-insert (partition)   
* 对某一特定列可以采用追加默认列的方式提高insert速度(特殊情况下)

##### 1.使用批量绑定bulk&#32;binding（显式游标批量处理）   
操作代码
```
DECLARE TYPE v_array IS TABLE OF VARCHAR2(20) INDEX BY BINARY_INTEGER;
    v_col1 v_array; 
    v_col2 v_array;
    v_col3 v_array;  
BEGIN 
    SELECT col1, col2, col3 BULK COLLECT INTO v_col1, v_col2, v_col3  FROM tab2;    
    FORALL i IN 1 .. v_col1.COUNT 
    insert into tab1  WHERE tab1.col1 = v_col1;  
END;  
```

* 优化原因：当循环执行一个绑定变量的sql语句时候，在PL/SQL 和SQL引擎(engines)中，会发生大量的上下文切换(context switches)。
* 使用批量绑定(bulk binding)能将数据批量的在plsql引擎与sql引擎之间传送，减少上下文切换过程中损失的切换代价。

##### 2.并行插入方法&#32;parallel-insert (partition)  
###### 采用并行插入的方法，为什么将并行插入（parallel-insert）和分区（partition）写在一起呢？  
* 谈到并行概念的时，不得不提到CS专业必考书《计算机操作系统》中的并行与并发概念。如果想了解二者区别，可以参考[这篇博文](https://blog.csdn.net/weixin_30363263/article/details/80732156)。强调并行操作，那么一定是多个CPU的或是一个多核CPU，因为一旦是单核且单个CPU便不可能实现并行，只能实现并发操作。
* 对单表实现并行操作会出现sequence排队现行，实则进行的仍是串行操作，后面等待中的并行操作以等待的方式一个接一个的操作表（串行）。如果表存在主键索引，随着主键递增，主键在多进程下索引分裂争用也是很严重的问题。
* 鉴于以上两点原因，因此对于单表和单个单核cpu，使用并行插入不会起到任何作用。

###### 并行具体操作 
> ##### 1.数据源为select全表扫描情况  
```
insert into tab1 select /*+ parallel */ * from table_name;  
commit;
```
* 作用：可加parallel的hint来提高其并发  
* 限制：最大并发度受初始化参数parallel_max_servers限制，此参数为数据库的最大并行度参数，值比已设置的过大，否则可能会导致并行要求超过数据库能力，原种情况下会导致数据库宕【dàng】机 。
* 可通过如下代码查看相关信息。  
```
SYS@prod>show parameter parallel_max_servers

NAME                      TYPE       VALUE
--------------------- ----------- ------------
parallel_max_servers    integer       40


SNAP_ID DBID INSTANCE_NUMBER PARAMETER_HASH PARAMETER_NAME  VALUE ISDEFAULT ISMODIFIED
----- ------ --- --------------- -------------------- ----- --------- -------
  11  434347367  1   3629921023  parallel_max_servers  40     TRUE     FALSE

......
......
......
  1  434347367   1   3629921023   parallel_max_servers  40    TRUE      FALSE
```
* 查看进程：并行进程在Oracle软件层面以v$px_session视图，在操作系统层面使用ps -ef |grep ora_p方式查看  

> ##### 2.插入数据为并行insert情况   

> #####并行insert操作代码：
```
alter session enable parallel dml;  

insert /*+ parallel */ into table2_name select * from table1_name;   

commit;  
```
* 在对表做DML并行操作之前，一定要打开你session级的parallel dml，在并行操作结束后将其关闭。
* 为什么要打开parallel dml，可以在执行计划中看到，在开启并行（parallel dml）在set autotrace explain后的执行计划后，update的操作步骤中有PX SEND QC （RANDOM）字样出现，证明DML操作使用到了并行操作。

4. 分区表并行insert操作 
``` 
insert into tab1 select * from table2_name partition (p1);    
insert into tab1 select * from table2_name partition (p2);    
insert into tab1 select * from table2_name partition (p3);    
insert into tab1 select * from table2_name partition (p4);  
```
* 对于分区表，可以利用多进程并行insert，同时对多个表进行插入；
* 分区越多、进程越多，批量操作时所花时间越少；
* 多分区可以配合上述一般解决方案中的弄logging方法加快速度。　 


> ##### 3.追加默认列提高insert速度（特殊情况下）
> ###### 操作代码
```
create table table1_name as select v1,v2 from table_name nologging；
alter  table table1_name add v_time date default sysdate not null;
```

1. 对于特定的表来说，可以使用CRAS(create table ... as select ..)方法创建表的部分数据，对于所更新列采用数据add default + not null的方法。  
2. 此种方法能够加快插入速率的原因是，在add默认列的时候，其实表没有真正的创建一真实物理列，而只是将数据更新到数据字典中。因此大大加快表的插入速率或更新速率。  
3. 12c和11g在默认列上有些许差异，详情可以查看这边博主的[博文](https://blog.csdn.net/conghe6716/article/details/100306917)



### 参考博文  
[参考博文1--parallel](https://blog.csdn.net/zhanglu0223/article/details/77869553)    
[参考博文2--parallel](https://www.cnblogs.com/nizuimeiabc1/p/4797110.html)  
[参考博文2--并发与并行](https://blog.csdn.net/weixin_30363263/article/details/80732156)   
[参考博文2--追加默认列](https://blog.csdn.net/conghe6716/article/details/100306917)