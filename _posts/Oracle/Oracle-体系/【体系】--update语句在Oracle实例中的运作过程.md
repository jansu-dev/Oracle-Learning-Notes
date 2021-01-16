---
title: 【体系】一条update语句在Oracle实例工作流程
date: 2019-10-13  
categories: Oracle
tags: Oracle体系
---


# 



![update执行图片](http://cdn.lifemini.cn/dbblog/20210115/24d3b906f2544f98a54210c40ee8f028.png)


### update数据法大致分为如下四个阶段：  
1. shared pool阶段 
2. buffer pool阶段
3. redo log buffer阶段
4. arch归档阶段

#### shared pool阶段
一条语句在SGA的shared_pool中的library cache中在parse阶段首先拿到执行计划， 
<font color="red">SQL语句执行六阶段：open-->parse-->bind-->execute-->fetch-->close</font>
在执行阶段(execute)将准备更新的最新数值(数据)写入database buffer cache中的缓存的数据块。

#### buffer pool阶段  
 1. buffer pool中已经存有需要修改的数据块缓存直接在buffer pool中缓存的数据块中刚修改

 2. buffer pool中没有需要修改数据的数据块缓存  
	
	通过server process将数据从硬盘I/O进bufferpool,  
	将数据I/O进buffer pool之前，需将基于旧时间点的数据所在的块存入undo表空间中，  
以便提供一致性读！！！ 
	
	在用户发起commit时，<font color="red">将会更新块头的scn</font>，即，将被修改的数据变为dirty buffer。  
	更新buffer cache中的undo块事务标记为已提交，此时undo表空间中的事务数据为历史数据，  
当下一个DBWR周期到来时，由于DBWR将dirty buffer写入datafile中，此时才算文件更新完成。
	
	DBWR进程在出发完全检查点的情况下，将被修改过的dirty buffer通过I/O写入datafile相应的数据块中。实现数据的持久化修改存储。

#### redo log阶段  

​	redo log buffer通过redo entries为单位记录了用户更新动作记录，  
​	以及server process将数据I/O进buffer pool中的动作。  
​	redo log buffer在LGWR进程的工作下，以串行的方式写入datafile中，  

​	一旦用户发起commit，将会立即触发完全检查点（ckpt），  
​	进到触发LGWR将日志条目从redo log buffer缓存写入磁盘redo log file中。

#### arch归档阶段   

​	随着redo日志不断写入，在当前日志组容量无法满足日志条目的存储时，切换当前日志组。

​	如果数据库为归档模式，ARCn开始工作将日志文件归档入预先设置好的数据库日志归档位置。  
