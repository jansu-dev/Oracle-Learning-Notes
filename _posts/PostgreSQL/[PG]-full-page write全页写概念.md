---
title:[PG]-full-page write全页写概念
date:2020-11-09
---



##### full-page write全页写概念



### 什么是full-page write

开启full-page write模式，服务端在数据页进行第一次物理I/O时将整个页面内容写到WAL或xlog日志中。这样在操作系统崩溃过程时，利用WAL日志中旧的物理页面数据，配合redo日志重放历史操作，达到恢复页面实例崩溃前的正确信息的目的。

### 什么时候发生full-page write

- 检查点之后
- 热备份开始之前