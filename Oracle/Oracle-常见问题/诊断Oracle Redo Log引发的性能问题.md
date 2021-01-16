---
title: 诊断Oracle Redo Log引发的性能问题
date: 2019-10-13
categories: Oracle
tags: Oracle常见问题
---




1. #### Rodo Log调整目标：

   在一个OLTP系统中Oracle尽量做到将大部分操作在内存中完成，以便最大程度的提升性能。  

​		因此在Oracle的诸多后台进程以及用户进程的大部分操作都是内存操作，而且这些操作会通过延迟写入技术尽可能的将磁盘I/O操作滞后。

​		但在在Oracle中针对Redo Log的操作由LGWR进程完成，LGWR是串行写入，一旦出现LGWR性能瓶颈将严重影响系统的性能，对数据安全也是潜在的威胁。 	

##### 			调整的目标：

​		做到Log_Buffer大小适中  

##### 			调整时机：

​		每当系统负载有一个明显的增加时，就应该考虑调整它的大小。

##### 			调整标准：

##### 			调整的目标：


>做到Log_Buffer大小适中  

##### 			调整时机：

>每当系统负载有一个明显的增加时，就应该考虑调整它的大小。

##### 调整标准：

> 不影响LGWR写日志文件的性能
> 不造成日志文件间的写入竞争
> 不在日志切换归档发生时引发磁盘竞争。

#####  调整手段：

> 日志文件的大小适中  
> 日志组的日志文件数量适中

2. #### 问题排查：  

> Redo Log问题监控两个方面：
> > 日志缓冲区空间使用的等待情况
> > 日志缓冲区数据槽的分配情况。

通过查询v$session_wait来监控日志缓冲区空间使用的等待情况
```shell
select sid,event,seconds_in_wait,state
from v$session_wait where
event='log buffer space%';
```
观察seconds_in_wait的数值来分析是否出现如下问题：
* 日志切换缓慢引发的等待
* LGWR写入缓慢引发的等待
* 日志文件写入引起的磁盘竞争引发的等待。

> 等待的发生的原因：
>
> > 日志文件写入时存在磁盘竞争,这种情况多见于日志切换发生时，  
> > 由于日志文件组的规划不当，或者存放日志文件的磁盘写入速度缓慢，  
> > 或者是因为磁盘RADI类型不当都会引发这个问题，如果怀疑村在这些情况，  
> > 可以通过如下语句进行监控：
```shell
select event,total_waits,time_waited,average_wait 
from v$system_event where event like k
'log file switch completion%';
```
观察total_waits，time_waited，average_wait数值是否过高
（注意何谓“过高”，不同系统考量标准不一要具体分析）  
##### sss等待问题解决措施：

* 将同一日志文件组的各个成员分配到不同的磁盘上，减少日志写入以及日志切换和日志归档时引发的竞争；
* 将日志文件尽可能存放在快速的磁盘上，推荐使用RADI10；
* 合理选择RADI类型对磁盘进行条带化；
* 增加REDO LOG文件大小延缓日志切换。



3. #### 换用等大的redo成员：
1. ALTER DATABASE ADD LOGFILE GROUP 4 (......)size 16M reuse;
2. ALTER DATABASE ADD LOGFILE GROUP 5 (......) size 16M reuse;
3. ALTER DATABASE ADD LOGFILE GROUP 6 (......) size 16M reuse;
4. alter system switch logfile;(多做几次，使旧的redo logfile成invalid)