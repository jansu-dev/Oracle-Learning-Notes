---
title: 【错误】--ORA-01555快照太旧及AUM介绍
date: 2019:10:22  
categories: Oracle
tags: Oracle错误
---



### ora-01555快照过旧错误

ora-01555错误在Oracle 8i的时候经常出现此错误，当数据库已经迭代到11g的时候，已经可以做到错误大量减少，但仍不可避免。  



### OERR提示 

```shell
[oracle@coresu ~]$ oerr ora 01555
01555, 00000, "snapshot too old: rollback segment number %s with name \"%s\" too small"
// *Cause: rollback records needed by a reader for consistent read are
//     overwritten by other writers
// *Action: If in Automatic Undo Management mode, increase undo_retention
//          setting. Otherwise, use larger rollback segments
```



### undo表空间中空间四种状态：

* active（活跃态）：表示事务还没有结束，为了提供多版本一致性读，undo表空间active态空间不可以被覆盖。
* expired（未到期态）：表示事务已经结束，包含commit和rollback的数据，在undo_retention保留期内Oracle尽力不然让数据被覆盖，但当时间实在不足时也可进行覆盖Oracle至今尽力做到不覆盖expired态空间中的内容。  
* unexpired（到期态）：表示事务已经结束，unexpired态空间中的数据可以随时被覆盖，unexpired是expired的下一个阶段，超过undo_retention时间的数据便从expired态进入到expired态。
* free（自由态）：自由态一般实在undo刚创建或拓展的时候出现。 

![状态关系图](http://cdn.lifemini.cn/dbblog/20210115/1cf3ed85dfe1436c9609dc675c8e10eb.png)


### ora-01555错误减少的原因
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在10g以前undo表空间采用的是手动管理，在undo已分配表空间容量被大量使用情况下，大量undo表空间数据处于expired态，即：undo_retention保留期内expired态数据不可被替换。此时，若事务量再次增加undo表空间已经无法提供足够空间，确保多版本一致性读功能，进而触发ora-01555错误。 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Oracle在10gR2版本中，引入了AUM（Auto Undo Management）自动管理undo表空间的概念，undo表空间已经是自动管理的，所以很少出现此错误。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以通过设置隐含参数_undo_autotune=false关闭AUT（Auto Undo Tune）自动undo表空间管理模式，但undo管理的自动化程度很高Oracle不建议关闭。



### AUM有两种状态工作方式：
1. autoextend off状态：此状态下忽略undo_retention的限制，参照undo表空间容量和undo相关统计信息计算出tuned_undoretention。此工作状态在undo表空间容量不合理时会引起undo警告，仍然可能触发ora-30036或ora-01555，贸然增大undo表空间的尺寸，又可能导致tuned_undoretention的增大。

2. autoextend on状态：此状态下以undo_retention为下限参考值，在tuned_undoretention时间内采用扩充undo表空间的方式，避免占用undo表空间中expired态的undo表空间。此种工作方式可能导致undo表空间不断扩充，而undo表空间又不会自动回收已经使用过的undo表空间，给DBA维护上带来麻烦。













