---
title: 【调优】Direct Path Reads产生原因及解决办法  
date: 2019-10-12
categories: Oracle
tags: Oracle调优
---



#### 目标：对大表的全表扫描时，通过Direct Path Read方式减少对于Buffer Cache的性能影响。

大表进行全表扫描时候：
| 版本 | wait event |
| :---:| :---: |
| oracle 11g以前 |  db file scattered read |
| 11g中 | direct path read |

> 优势：
> * 减少了对栓latch的使用，避免可能的栓争用
> * 物理IO的大小不再取决于buffer_cache中所存在的块

(假设某个8个块的extent中1,3,5,7号块在高速缓存中，
而2,4,6,8块没有被缓存，传统的方式在读取该extent时将会是对2,4,6,8块进行4次db file sequential read,此效率比单次读取这个区间的所有8个块还要低很多，
虽然Oracle为了避免这种情况总是尽可能的不缓存大表的块(读入后总是放在队列最冷的一端);
而direct path read则可以完全避免这类问题，尽可能地单次读入更多的物理块。) 


> 缺点：
> * 在直接路径读取某段前,需对该对象进行一次段级的检查点.
> * 可能导致重复的延迟块清除操作。
> * 如果全表扫描操作频繁，会导致IO问题引起宕库。



#### 诊断步骤：
1.通过如下SQL查询当前等待事件，或这从sp报告中获取。 

```shell
select event, count(1)from v$session_wait 
WHERE EVENT NOT IN
(select E.NAME from V$EVENT_NAME E WHERE E.WAIT_CLASS = 'Idle') 
group by event 
order by 2 desc;
```

2 找出产生direct path read的SQL语句 

```shell
select sql_id, count(1)
  from v$session
 where event = 'direct path read'
 group by sql_id;

select * from v$sql
where sql_id in (select distinct sql_id
from v$session where event = 'direct path read');
```



#### 参数

| :----: | 11GR1 |  11GR2 |  备注  |
| :---- | :----- | :---- | :---- |
| Block cache阀值 | 50% | 50% | 少于此阀值 |
| 脏块阀值 | 25% | 25% | 少于此阀值 |
| 块阀值 | _small_table_threshold*5 | _small_table_threshold' | 统计信息里记录的表的block数目（11GR2)超过此阀值。 |

特别注意第一个阀值，参照的是统计信息里的表的block数，而不是dba_segments里的。如果你的统计信息不准确，很可能导致扫描路径变为direct path read（统计信息值过大）,或者怎么都走不上direct path read（统计信息值过小）

设置这三个阀值非常的合理：

第一个阀值：表大小，太小的表从direct path read中的获益太小

第二个阀值：表在内存里的cache率,如果cache率很高，那么还是走传统路径更快

第三个阀值：脏块阀值，由于direct path read需要出发一个段的检查点，因此脏块太多，刷新脏块可能会导致IO繁忙


ORACLE似乎对脏块的比例比较敏感，能立即修正执行计划，但是对block cache的变化不敏感，有时候表的大部分数据已经被cache了，但是执行计划依然会选择走dpr

隐含参数：
_small_table_threshold
如果表大于 5 倍的小表限制，自动会使用DPR替代FTS。

初始化参数： _serial_direct_read 
禁用串行直接路径读。

Oracle通过一个内部的限制，来决定执行DPR的阈值。
可通过10949事件屏蔽这个特性，返回到Oracle 11g之前的模式上： 

```shell
alter session set events '10949 trace name context forever, level 1';
```

参数: _very_large_object_threshold（MB单位）
用于设定使用DPR方式的上限，这个参数需要结合10949事件共同发挥作用。

10949 事件设置任何一个级别都将禁用DPR的方式执行FTS，  
但是仅限于小于 5 倍 BUFFER Cache的数据表，  
同时，如果一个表大于 0.8 倍的 _very_large_object_threshold设置仍会执行DPR。

