---
title:[TiDB]-监控关键信息
date:2020-12-27
---



#### [TiDB]-监控关键信息



### 关键指标

Overview--TiDB
Duration：SQL 响应时间，按照百分位计算方式；

Connection Count:连接数，关注tidb-serer是否均衡以及总连接数是否符合预期；

PD TSO Wait Duartion：从PD Server获取等待时间，正常情况下99百分位的值应该在ms级别；
最简单的SQL，都会超多这个时间；

Lock Resolve OPS:锁冲突的数量，如果这个指标的数值达到两位数，说明存在较为严重的冲突情况；

Overview--TiKV

leader & region ：主要看各tikv实例分布是否均匀，可能跟pd调度或tikv参数配置有关；

CPU：这里是按照单个core计算的，也就是会出现超过100%的情况，重点关注使用率是否合理，以及是否存在热点状况，即各tikv实例的CPU消耗不均衡；

channel full;出现channel full意味着对应的模块出现了拥堵情况，许哟啊视u提情况分析；

server report failures:正常的服务调用返回的错误，例如实例之间的心跳丢失unreachable等；

scheduler pending commands: scheduler负责写入相关，如果出现 scheduler pending意味着写堆积，比较常见的是写入热点引起的scheduler pending情况，也可能是资源出现瓶颈等情况。

TIKV-details

coprocessor executor count & request duration:与scheduler模块相应，coprocessor主要负责处理读请求，根据这两个指标可以看当前集群处理的具体情况。

raft store CPU：raft store模块主要负责处理raft相关请求，例如写入时raft log分发，以及raft group的心跳请求等等，在3.0版本默认最多使用2的线程，所以则例的上限是200%,如果发现raft store CPU消耗超过150%，可能意味着raft请求处理成为瓶颈，常见的可能是写入过多或者是regin过多，需要具体分析，同时这里也判断是否出现热点的关键指标；

coprocessor CPU:跟上面类似，只是这里主要针对的是请求的处理，默认是CPU store数的80%上限，所以偏AP的场景需要更多的CPU核数，同时这里也是判断是否出现都热点的关键指标；

gRPC ：

Overview--pd

leader balance Ratio: leader适量最多和最少相差百分比，一般在5%以内；

Region Balance Ratio:region数量最多和数量最少相差的百分比，一般在5%以内；

Region Health:如果出现某个实例当即，这里会表明多少各region受影响，比如缺副本数等；

Hot Write/Read region's leader distribution:写/读热点region的分布情况；



### 告警的级别

tidb-server级别
warning

NODE_cpu_used_more_than_80%(warning)

NODE_tcp_estab_num_more_than_50000(warning)

磁盘读和谐的延迟，在跑AP的SQL时候I/O的压力可能比较大可以降低一定的标准；
NODE_disk_read_latency_more_than_32ms(warning)

NODE_disk_write_latency_more_than_16ms(warning)

ermergency

TiDB_schema_error(emergency):获取元数据信息失败，可能会造成tidb不可用

十秒内如果tikv错误超过6000告警
TiDB_tikvclient_region_err_total(emergency):increase[10m]>6000

TiDB_domain_load_schema_total(emergency):increase{faild}[10m]>10

TiDB_monitor_keep_alive(emergency):increase[10ms]<100

TiDB_server_panic_total(critial):TiDb server panic

TiDB_memory_abormal(warning):TiDB heap memory usage is over 10GB

TIDB-server级别

TiDB_query_duration(arning):99百分位duration>1s

TiDB_server_event_error(warning):increase (server_start|server_hang)[15m]>0

TiDB_tikvclient_backoff_count(warning):increase[10m]>10

TiDB_monitor_time_jump_back_error(warning):time_jump_back

TiDB_ddl_waiting_jobs(warning):sum(waiting jobs)>5

critical

tikv-server级别：