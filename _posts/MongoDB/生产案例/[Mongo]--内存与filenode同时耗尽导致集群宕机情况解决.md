---
title:[Mongo]--内存与filenode同时耗尽导致集群宕机情况解决
date:2020-09-27
---



​		被开发人员通知mongo集群再次宕机，经排查发现是由于内存与filenode同时耗尽导致的。

### 集群宕机原因分析

**节点二（从节点）宕机原因**
节点二于昨天晚上22点，由于内存占用过多被系统杀掉。

日志关键信息：

```
Sep 23 22:00:25 P-CC-Mongo2 kernel: Out of memory: Kill process 23482 (mongod) score 957 or sacrifice child
Sep 23 22:00:25 P-CC-Mongo2 kernel: Killed process 23482 (mongod) total-vm:25268876kB, anon-rss:7307420kB, file-rss:0kB, shmem-rss:0kB
```

**节点一（主节点）宕机原因**
主库于今天下午15:02:46宕机，日志报错Too many open files，由于并发和mongo自身引擎机制，超过linux最大限度导致主节点宕机。

日志关键信息：

```
2020-09-24T15:02:46.124+0800 E STORAGE  [conn4129] WiredTiger error (24) [1600930966:124840][22700:0x7fdf64063700], 
        WT_SESSION.create: __posix_directory_sync, 160: /data/db/mongo/data/hxcc/in
        dex-2273687--801829927859739304.wt: directory-sync: Too many open files 
        Raw: [1600930966:124840][22700:0x7fdf64063700], WT_SESSION.create: __posix_directory_sync,
        160: /data/db/mongo/data/hxcc/index-2273687--801829927859739304.wt: directory-sync: Too many open files
```



### 宕机原因定论

主节点宕机的前一天晚上从节点因内存占用问题宕机，主节点于当日下午因Too many open files主节点宕机，主从全部宕机导致业务无法正常运行。



### 集群宕机造成的后果

主库与备库日志间隔18.19小时，无法及时自动回复，存在集群不同步现象。

日志关键信息：

```
日志验证：repset:PRIMARY> db.printSlaveReplicationInfo()
          source: 10.5.80.128:27017
                syncedTo: Wed Sep 23 2020 22:00:24 GMT+0800 (CST)
                65477 secs (18.19 hrs) behind the primary
```



### 补救措施

1. 在非业务时间晚上手动重新初始化集群，使得集群主从节点同步。
2. 在主节点和从节点添加内存限制参数，避免再次出现因内存问题导致的宕机。
3. 继续增大主从节点的inode最大限制（nodefile），当前值为204800，参照file-max系统理论最大可设为1608191参考值，本次准备将值修改为9000000。

查看最大限制

```shell
[XXX_user@XXXX-Mongo1 usr]$ cat /proc/sys/fs/file-max
```