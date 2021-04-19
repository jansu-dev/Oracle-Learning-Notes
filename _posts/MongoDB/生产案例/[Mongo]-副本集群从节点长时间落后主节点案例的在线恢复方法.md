---
title:[Mongo]-副本集群从节点长时间落后主节点案例的在线恢复方法
date:2020-09-28
---



​	手里新接手了一些mongo数据库，进行节巡检时发现有一个集群在2018年就已经宕机了，当前时间为2020年。



### 问题描述

主库查看集群同步情况如下
主节点日志：

```
repset:PRIMARY> db.printSlaveReplicationInfo();
source: 10.16.25.214:27017
    syncedTo: Sun Sep 27 2020 10:39:48 GMT+0800 (CST)
    0 secs (0 hrs) behind the primary 
source: 10.16.25.35:27017
    syncedTo: Mon Aug 03 2020 10:07:47 GMT+0800 (CST)
    4753921 secs (1320.53 hrs) behind the primary
```

从节点检查，发现从库节点宕机。

```shell
[root@XXX_25_35 mongo_log]# ps -ef|grep mongo
root     26023 23086  0 10:45 pts/2    00:00:00 grep --color=auto mongo
```

重启从节点后，在主库检查如下。

```
repset:PRIMARY> db.printSlaveReplicationInfo();
source: 10.16.25.214:27017
    syncedTo: Sun Sep 27 2020 10:10:07 GMT+0800 (CST)
    0 secs (0 hrs) behind the primary 
source: 10.16.25.35:27017
    syncedTo: Mon Aug 03 2020 10:07:47 GMT+0800 (CST)
    4752140 secs (1320.04 hrs) behind the primary
```

但是，启动后备库检查，一直处于RECOVERY状态。
![1.png](http://cdn.lifemini.cn/dbblog/20200927/3bf826ea3454426886627c11cbc79dec.png)



### 解决办法

在线重新初始化副本从节点，这是恢复从库可以将从库重新初始化：
1.登陆从库，关闭从库；
2.将从库dbpath对应的目录MV改名，目的是得到一个空的dbpath；
3.重启从库，从库会自动重新全量同步，重新初始化期间使用rs.status()可以看到从库的stateStr为STARTUP2，初始化结束后会变回SECONDARY。
![2.png](http://cdn.lifemini.cn/dbblog/20200927/d8c30127f9724a15bc8cc220dea126dd.png)

操作步骤

从库端：

```
repset:RECOVERING> db.shutdownServer()
assert failed : unexpected error: Error: shutdownServer failed: shutdown must run from localhost when running db without auth
Error: assert failed : unexpected error: Error: shutdownServer failed: shutdown must run from localhost when running db without auth
    at Error (<anonymous>)
    ......
    ......
    at assert (src/mongo/shell/assert.js:20:5)
    at DB.shutdownServer (src/mongo/shell/db.js:212:9)
    at (shell):1:4 at src/mongo/shell/assert.js:13
repset:RECOVERING> exit
bye
[root@XXX_25_35 mongo_log]# ps -ef|grep mongo
root     23394     1  0 10:04 ?        00:00:16 /home/db/mongodb/bin/mongod -f /home/db/mongodb.conf
root     25948 23086  0 10:44 pts/2    00:00:00 grep --color=auto mongo




[root@XXX_25_35 mongo_log]# cd /home/db/
[root@XXX_25_35 db]# mv db_data db_data_20200927
[root@XXX_25_35 db]# mkdir db_data



[root@XXX_25_35 db]# /home/db/mongodb/bin/mongod -f /home/db/mongodb.conf
note: noprealloc may hurt performance in many applications
about to fork child process, waiting until server is ready for connections.
forked process: 26086
child process started successfully, parent exiting
```

从节点加入集群时状态：
自动恢复

![3.png](http://cdn.lifemini.cn/dbblog/20200927/4c1002e2bd944e9eaee92bbe79d7b235.png)

主库恢复正常

```
repset:PRIMARY> db.printSlaveReplicationInfo();
source: 10.16.25.214:27017
    syncedTo: Sun Sep 27 2020 11:05:18 GMT+0800 (CST)
    0 secs (0 hrs) behind the primary 
source: 10.16.25.35:27017
    syncedTo: Sun Sep 27 2020 11:05:18 GMT+0800 (CST)
    0 secs (0 hrs) behind the primary
```



### 问题总结

日常巡检或脚本监控很重要！！！



### 参考文章

- [csdn--遇星的文章](https://blog.csdn.net/weixin_39004901/article/details/102597400)

- [博客园--散尽浮华的文章](https://www.cnblogs.com/kevingrace/p/8178549.html)