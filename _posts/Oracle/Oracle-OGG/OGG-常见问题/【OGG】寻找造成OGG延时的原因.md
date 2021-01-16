### 文章概览
- [Oracle数据库版本](#Oracle数据库版本)  
- [是什么？](#是什么？) 
- [为什么？](#为什么？)  
- [怎么做？](#怎么做？)  
- [参考文章](#参考文章)

### Oracle数据库版本  
&nbsp;&nbsp;&nbsp;&nbsp;操作系统：   
&nbsp;&nbsp;&nbsp;&nbsp;OGG版本号：   
&nbsp;&nbsp;&nbsp;&nbsp;Oracle数据库版本：


### 是什么？  
&nbsp;&nbsp;&nbsp;&nbsp;由于工作中经常运维OGG，所以会遇到OGG延迟的问题。   
&nbsp;&nbsp;&nbsp;&nbsp;此篇文章讲述如何查找OGG延迟的原因。

### 为什么？
1. 首先我们来介绍一下指标LAG；  
    - 首先，OGG的检查点进程在数据库事务结束之后，将事务信息写入online redo logfile中，再由Extract进程读取事务信息至TRAIL或RMTTRAIL中，最后被DUMP和Replicat进程处理。  
    - 在数据库写入日志时间点到被EXTRAIL进程处理之间，存在时间差，这个时间差就是表示还未被OGG处理的时间差，LAG是一种OGG延迟指标。  
    - 注意不同LAG表示的意义不同：  
        + 取自info的LAG，LAG信息来自于MANAGER进程中记录的checkpoint处理时间点（单位/时间）。
        + 取自send的LAG，LAG信息来自于被Replicat进程正在处理的时间戳（单位/Kilobytes）。
2. LAG是衡量OGG延迟的指标，那么当LAG指标出现几个小时的问题时，就说明OGG延迟比较高。  
3. 造成OGG延迟的问题很大一部分原因是，存在大量的事务未被目标端REPLICAT进程处理，具体以什么原因造成此问题的原因还需要仔细排查。

### 怎么做？
- 查看事务信息  
查看MANAGER中记录的LAG信息
```
info all
```
查看Replicat进程处理LAG信息
```
send 进程名,status
```


- 目标端查看长事务 
    + 依据Linux命令查看REPLICAT进程SPID
```
ps -ef |grep repfull
```
    + 依据进程ID抓出SQL_ID  
```
SELECT sql_text,sql_id FROM v$sqltext a 
WHERE a.hash_value = (SELECT sql_hash_value FROM v$session b 
WHERE b.SID =(select s.sid from v$session s,v$process p 
where s.paddr=p.addr and p.spid='REPLICAT进程ID')) ORDER BY piece ASC;
```

    + 依据SQL_ID查看执行计划
```
select plan_table_output
 from table(dbms_xplan.display_cursor('SQL_ID'));  
```
- 最后，再对具体SQL进行具体的处理工作。  

- 造成OGG延迟的原因有很多，大体归为如下几类：
    - 源端E进程abend导致pump和Replicat进程均出现延迟；
    - 源端大表批量更新操作（操作几千万上亿条数据），大量同步事务执行过慢；
    - 表无主键或唯一索引，导致同步更新时全表扫描，同步事务过慢；
    + 表统计信息过旧，导致同步事务执行过慢；
    - Replicat进程里面的表过多，处理不过来；
    + MGR管理进程挂死；
    + 网络中断或主机故障，导致目标端与源端通讯异常；



- 几种简单的解决方案：

    - Extract进程配置COMPRESS，OGG以1:4压缩比例压缩trail文件。
    - 单进程出现热点表，拆分多进程。
    - 如果是单张表出现热点问题，可以基于值操作，将同一张表不同数据按值划分到不同进程中。

- 一种临时性解决方案：  
    + 当OGG进程挂掉后，查出出问题的RBA暂时跳过，可直接运行下一个RBA以支持业务。
    + 但此种方法不是长久之计，因为问题依旧存在，应将问题找出再解决。

### 参考文章 
- [参考文章1-理解LAG](https://tieba.baidu.com/p/3791421261?red_tag=1308070587)
- [参考文章2-网络层优化](https://blog.csdn.net/zwjzqqb/article/details/81165433)
- [参考文章3-拆分进程操作步骤](https://www.cnblogs.com/zjpeng/p/10881829.html)
- [参考文章4-统计信息过慢导致延迟](https://blog.51cto.com/yangjunfeng/1944421)
- [参考文章5-OGG跳过出错的RBA](https://www.cnblogs.com/hxlasky/p/10261206.html)