---
title:[PG]-通过修改autovacuum触发机制更好的清理表空间垃圾
date:2020-11-09
---



##### 通过修改autovacuum触发机制更好的清理表空间垃圾

1. 通过修改autovaccum_max_workers参数增加与PG Cluster的数据库等数量的autovacuum工作进程数量，可以在同一时刻并行进程vaccum处理。数据库存在于表空间下，因此清理数据库可以间接更好的清理表空间垃圾。
2. 如果数据库服务器资源有限，并且I/O工作量大，调整autovaccum_vaccum_cost_limit（默认值200个数据页）限制一次工作的资源最大使用量。
   通过调整autovaccum_vaccum_cost_delay参数，达到最大成本限制时调整工作进程的休息时间。
   通过调整autovaccum_cost_page_hit参数，调整在清理时计算autovaccum时发生逻辑读的成本计算参考值。
   通过调整autovaccum_cost_page_miss参数，调整在清理时计算autovaccum时发生物理读的成本，也就是真实磁盘I/O的成本计算参考值。
   通过调整a utovaccum_cost_page_dirty参数，调整在清理时计算autovaccum时发生死元组的写入成本计算参考值。
   结合各个参数精细控制autovaccum的执行过程，具体如何调节依据业务高峰时间及数据库硬件资源和DBA经验调整，达到更好的清理表空间垃圾的目的。
3. 也可以在表级调整autovaccum_vaccum_cost_limit、autovaccum_vaccum_cost_delay、autovaccum_vaccum_scale_factor微调大表的autovaccum。