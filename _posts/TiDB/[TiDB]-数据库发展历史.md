---
title:[TiDB]-数据库发展历史
date:2020-12-20
---



#### [TiDB]-分布式数据库发展历史



### **存储引擎（Stroge engine）**

- **B-Tree**
  InnoDB
  LMDB
  WiredTiger
- **LSM-Tree**
  LevelDB
  RocksDB
  WiredTiger
- **Others**
  BW-Tree（微软发的paper）
  TokuDB（分型树技术）
  WiscKey（LSM-Tree优化）

**Paper与项目发展史**

Paper

- Google
  - GFS
  - MapReduce
  - BigTable
- Amazon
  - Dynamo
- Apache Cassandra
  - WRN



### 项目

- MongoDB
  - 非结构化数据大大提高用户易用性
- ElasticSearch
  - 验证了搜索方面的需求
- Memcached/Redis
  - 使用内存处理缓存需求

**OLAP & Data warehouse**

- Hive
  - Hive是计算和分析的引擎，实际数据存爱hdfs。
  - MapReduce分布批处理。
  - 缺点是需要多次存储中间结果。
- Impala/Kudu
  - Impala是用内存优化后的计算和分析的引擎。
- CreenPlum
  - GP基于PG做成了一个分布式计算引擎。
  - 自己开发了一个强大的优化器。
- Apache Kylin
  - 将原始数据进行预计算，对报表类项目支持特别好。
- Apache Spark
  - 独立于存储层，可以接入不同存储，不局限于hdfs。
  - 想要替换MapReduce。



### **newSQL**

- Spanner
- Tidb
- Auwa