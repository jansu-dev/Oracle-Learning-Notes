---
title:[TiDB]-五大重要特性与四大应用场景
date:2020-12-20
---



#### [TiDB]-重要特性与应用场景

- [**TiDB五大特性**](**TiDB五大特性**)
  - [一键水平扩容或者缩容](一键水平扩容或者缩容)
  - [金融级高可用](金融级高可用)
  - [实时HTAP](实时HTAP)
  - [分布式云原生数据库](分布式云原生数据库)
  - [兼容 MySQL 5.7 协议和 MySQL 生态](兼容 MySQL 5.7 协议和 MySQL 生态)
- [**TiDB 4大核心场景**](**TiDB 4大核心场景**)
  - [对数据一致性及高可靠、系统高可用、可扩展性、容灾要求较高的金融行业属性的场景](对数据一致性及高可靠、系统高可用、可扩展性、容灾要求较高的金融行业属性的场景)
  - [对存储容量、可扩展性、并发要求较高的海量数据及高并发的 OLTP 场景](对存储容量、可扩展性、并发要求较高的海量数据及高并发的 OLTP 场景)
  - [Real-time HTAP 场景](Real-time HTAP 场景)
  - [数据汇聚、二次加工处理的场景](数据汇聚、二次加工处理的场景)



### TiDB五大特性

**一键水平扩容或者缩容**

由于存储及计算分离，TiDB可对运维人员透明扩容。

**金融级高可用**

存储多副本、Raft 分布式协议、多数派提交满足强一致性；
可接副本数量、地理位置配置容灾。

**实时HTAP**

行存TiKV；
列存TiFlash 通过 Multi Raft leaner 实时副制数据,进而保证TiKV和TiFlash 数据一致（强一致）；
TiKV和TiFlash 可按需配置于不同机器，解决资源隔离问题。

**分布式云原生数据库**

通过 TiDB Operator 可在公有云、如有云、混合云中实现部署具化，自动化。

**兼容 MySQL 5.7 协议和 MySQL 生态**



### TiDB 4大核心场景

**对数据一致性及高可靠、系统高可用、可扩展性、容灾要求较高的金融行业属性的场景**

高可靠
系统高可用
可扩展性
高容灾的金融场景

传统：
同城两机房服务、异地一机房不服务，资源利用率低、RPO与RTO不理想；

(Recovery Point Objective)复原点容忍最大数据丢失量

（可容忍数据最大铁量1
TB 逼过瓜以1班多副本解决

(Recovery Time objective)复原时间目标,即：可容忍服务中断时间长度
RTosos.in =0

**对存储容量、可扩展性、并发要求较高的海量数据及高并发的 OLTP 场景**

② 大容量、高并发，可扩展的以印数据库场景
传统： 采用分库分表中间件、
高性脂硬件(Ran
下口只属于 New SQL 解决方案（计算存储分离解决
计算最大 mode 数512
敏 mode 最大致 no

**Real-time HTAP 场景**

② Real-Time HT Ap 场景 （容量最大支持 PB 级别
传统。通过正儿同步至10 LAP DB 成本高.
1实时性差
TDD:𥑆340通过下 Flash 结合行存和列存实现实时 MAP, ，
实现同一个1713同时兼具0 UP. OLAP.

**数据汇聚、二次加工处理的场景**

④ 数据汇聚、二次加工场景
传统、采用砒十 Hadoop 完成。
下图13：通过此工具或下𠮨同步二是同步及不𠮨自动生成报表
下口1340新特性
① 调用热度1写入1读取量维度
kg 维度
② 存储引擎 增加到存不 Flash
4-0下，比提供新的存储格式
提升表场景下的编解码数据效率
② 下 DB Dash body DBA 可通过 Dashboard 各项指标了解分析系统。
duster info i 提供不 DB.TW. PD Ii Flash.主机运行状态。
| key visualize r:年统可视化输出下𠮩集群一段时间内的流量情况。
作用分析集群使用模式
② 排查流量热点
fl shunt:记录处统计信息，文本、执行次数、执行时间汇总等
作用：排查热点 SQL
Slow Queries..汇总集群中所有𪚩询 SQL
(Diagnostic Report: 周期性对集群诊断 （类 awri
Log Search & Download 集群日志信息，
④ 部署运维工䳼管理
集群管理
本地部署
|和有镜像管理
性能测式
① 轍|悲观事务的，
支持 Read committed
select for update no wait
（支持大事物，最大轍持心比此前为心 MD
⑥ SQL 功能
㦖䲜沙
了， select into outfit 及 Load data
4.支持 Sequence
f 纛䲜！靠近时，趴数据至磁盘(join, sort,
7， 墈口 explain, explain analyze
8支持 index merge.（单表多 index 组合粳用）
| 9-新增信息于 information schema g cluster into 集郡拓扑信息

- clustering 系统日志
  cluster_hardware 了硬件信息，操作享统信息
  （ 1 cluster_system into
  W. f he slow_query 全局憒晓，
  cluster_process list 全局 process 皉
  | inspection_result 4.0自动性能诊断
  mertics-summary­mertics_summary_hghabeB.sn 监控
  (inspection summary 保存数据链路信息

⑦ 妊
大小尔喑不敏感排序 uhnb-general.ci the genera Ld
⑧ 舭备份恢复
⑨ 服务级别功能
f 支持缓存 Prquekxe.ua 㵾的执行计划
支持自适应线程池
Follower Read 强故读功能下， Region 的 follower 副本读功能
④ Ti CDC 变更装推送至其它中间件
基本功能 分区新 hash 分区
Range 分区
视图普通视图
非𧪾
徚| 主键约来
唯一约束
兼副生问题。自增土口|阳陌增列仅限于单个：D B Server,（混用自定义值和自增列报错 Duplicate Emi
下吅通过 tidh-ahow-remeauto.in 自增列属性 fur table modify
② SQL Statement 最大限制默认5000，（可通过 stmt-ount.tn.䉮鬱 table change