---
title:[Mongo]--因超过Linux最大inode数导致集群宕机问题锁定与解决
date:2020-08-21
---



因超过Linux最大inode数导致集群宕机问题锁定与解决。

![案例.jpg](http://cdn.lifemini.cn/dbblog/20200821/4b97eab598e44cb795bdd8fd5828468a.jpg)

### 环境信息

MongoDB版本：3.6.17
操作系统：centos7.8
集群方式：主 + 从 + 仲裁
  主节点：10.5.80.127:27017
  从节点：10.5.80.128:27017
  仲裁节点：10.5.80.128:27018



### 问题描述

下午，突然接到开发通知，某客户mongo集群宕机了。
查看发现，主节点宕机、从节点和仲裁节点均正常运行，为了抓紧恢复业务匆忙起库。
起库后，虽然业务短暂恢复，可如下问题随之产生。

1. **导致主节点宕机的原因。**
2. **从节点没有接管业务的原因。**



### 关键信息

主节点日志信息：

```
2020-08-19T13:14:48.788+0800 E STORAGE  [conn104] WiredTiger error (24) [1597814088:787413][7206:0x7f408d3f0700], WT_SESSION.create: __posix_directory_sync, 145: /data/db/mongo/data/hxcc/: directory-sync: open: Too many open files Raw: [1597814088:787413][7206:0x7f408d3f0700], WT_SESSION.create: __posix_directory_sync, 145: /data/db/mongo/data/hxcc/: directory-sync: open: Too many open files
2020-08-19T13:14:48.789+0800 E STORAGE  [conn104] An unsupported journal format detected - If you are trying to rollback from version 4.0 to 3.6, please re-start a 4.0 binary and cleanly shut it down so that the journal format will be downgraded.
2020-08-19T13:14:48.789+0800 E STORAGE  [conn104] WiredTiger error (24) [1597814088:789199][7206:0x7f408d3f0700], WT_SESSION.create: __posix_directory_sync, 160: /data/db/mongo/data/hxcc/collection-65338--4884278444460407547.wt: directory-sync: Too many open files Raw: [1597814088:789199][7206:0x7f408d3f0700], WT_SESSION.create: __posix_directory_sync, 160: /data/db/mongo/data/hxcc/collection-65338--4884278444460407547.wt: directory-sync: Too many open files
```

mongo主节点日志中，关键信息directory-sync: open: Too many open files说明了问题所在。
经了解得知，mongo在3版本以后采用WiredTiger引擎替换Mmap引擎，该版本引擎给每张表和每个索引创建一个文件；并且创建完毕不是立即关闭，在系统层面文件一直处于打开状态，如果mongo引擎未及时关闭并累加到系统限制值就会宕机。

**最终确定，本次主节点宕机的原因是WiredTiger引擎超过linux系统最大文件数限制导致的宕机。**

从节点日志信息：

```
2020-08-19T15:59:49.883+0800 I REPL     [rsBackgroundSync] sync source candidate: 10.5.80.127:27017
2020-08-19T15:59:49.883+0800 I REPL     [replication-0] We are too stale to use 10.5.80.127:27017 as a sync source. Blacklisting this sync source because our last fetched timestamp: Timestamp(1597143607, 2407) is before their earliest timestamp: Timestamp(1597798788, 7) for 1min until: 2020-08-19T16:00:49.883+0800
2020-08-19T15:59:49.883+0800 I REPL     [replication-0] could not find member to sync from
2020-08-19T16:00:20.639+0800 I NETWORK  [LogicalSessionCacheRefresh] Starting new replica set monitor for repset/10.5.80.127:27017,10.5.80.128:27017
```

至于，为什么从节点副本在运行状态下没有接管业务继续提供数据库服务。

1. We are too stale to use 10.5.80.127:27017 as a sync source.
2. could not find member to sync

在从节点日志中，从两条关键信息可以看出从节点一直处于无法与主节点同步状态。

主节点查看同步状态，db.printSlaveReplicationInfo()发现从节点落后主节点248.1个小时,进一步确认了两节点不同步情况。
同时，rs.status()发现从节点一直处于RECOVERING状态。

```
PRIMARY> db.printSlaveReplicationInfo();
source: 10.10.10.2:110
        syncedTo: Tue Aug 19 2020 17:25:04 GMT+0800 (CST)
        10 secs (248.1 hrs) behind the primary 
source: 10.10.10.3:27018
```

**最终确定，本次从节点在主节点宕机宕机之后没有接管业务的原因是从节点与之前提供业务的主节点同步相差较大，存在数据不一致情况，所以无法接管业务**



### 解决方案

1. 在修改/etc/security/limits.conf文件中，通过调整hard nofile及soft nofile，增大该用户可以使用的最大文件数解决Too many open files引发的宕机问题。

1. 重新同步从节点数据，使从节点与主节点一致。

操作步骤：
1、 备份主节点所有的数据；
2、 停止mongodb服务；
3、 将从节点数据存放数据的data文件价重命名，并创建新的data空文件夹；
4、 将三台mongodb服务启动；
5、 从节点会从主节点同步数据。

```
/data/db/mongo/mongodb/bin/mongodump -d admin -o /data/mongobak/admin_20200820.dmp
/data/db/mongo/mongodb/bin/mongodump -d XXX -o /data/mongobak/XXX_20200820.dmp
/data/db/mongo/mongodb/bin/mongodump -d local -o /data/mongobak/local_20200820.dmp
......
......


--三个节点均操作
use admin
db.shutdownServer()

--仅从节点操作
mv ./data ./data_bak_20200820

--三个节点均操作
/data/db/mongo/mongodb/bin/mongod -f /data/db/mongo_cluster.conf
```

检查状态：

```
rs.status();

db.getReplicationInfo();

db.printSlaveReplicationInfo();
```

最后发现，从节点状态已从RECOVERING转变为了SECONDARY，集群恢复正常。

<span style='color:red'>**注意：**</span>

<span style='color:red'>1. 理论来说，主节点启动后，从节点会自动回滚数据恢复，但此次从节点未自动追平，猜测原因为距离主节点时间太远导致。</span>
<span style='color:red'>2. 理论来说，可以不用全部停止整个集群服务重新初始化从节点数据，但处于安全考虑，本次操作还是选择了晚上停止所有数据库服务操作。</span>