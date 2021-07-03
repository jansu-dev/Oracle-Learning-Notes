---
title:[Mongo]--集群搭建（副本集）
date:2020-06-23
---



##### mongo集群搭建（副本集）

### 节点一

```
mkdir -p /home/user/db/mongo
mkdir -p /home/user/db/log
mkdir -p /home/user/db/data

tar -zxvf mongo....tar.gz -C /home/user/db/mongo

mv /home/user/db/mongo/mongo.... /home/user/db/mongo/mongodb

vi /home/user/db/mongo/mongodb_jiqun.conf

--数据文件目录
dbpath=/home/db/mongo/data
--日志文件目录
logpath=/home/db/mongo/log/mongodb.log
--每个数据库数据文件对应一个目录
directoryperdb=true
--多次启动时日志采用追加
logappend=true
--副本集名称
replSet=repset
--绑定IP,
bind_ip=0.0.0.0
--绑定端口
port=27017
--oplogsize最大可用大小(单位MB)
oplogSize=1024
--后台运行子进程
fork=true
--预分配数据文件
noprealloc=true
--对于集群来说的认证文件
#keyFile=/home/db/mongo/data/keyfile
--是否开启认证
#auth=true
--wiredTiger引擎可用内存大小
wiredTigerCacheSizeGB = 36



--启动节点
/home/user/db/mongo/mongodb/bin/mongod -f /home/user/db/mongo/mongodb_jiqun.conf
```

<span style='color:red'>注意：</span>
<span style='color:red'>1. mongod首次创建OPLOG后，改变oplogSize将不会影响OPLOG的大小。</span>
<span style='color:red'>2. 启动之前，配置文件中目录要提前存在。</span>
<span style='color:red'>3. mongo3.6以上版本，支持oplog动态扩容，仍需每个节点单独操作。3.6版本以下，需要脱离集群重新初始化oplogsize大小。</span>



### 节点二

参照节点一，在节点二的27017和27018两个端口，以副本集配置文件启动两台mongo实例。

```
--配置文件一
dbpath=/home/user/db1/mongo1/data
logpath=/home/user/db1/mongo1/log/mongodb1.log
directoryperdb=true
logappend=true
replSet=repset
bind_ip=0.0.0.0
port=27017
oplogSize=1024
fork=true
noprealloc=true
#keyFile=/home/user/db1/mongo1/data/keyfile
#auth=true
wiredTigerCacheSizeGB = 12


--配置文件2
dbpath=/home/user/db2/mongo2/data
logpath=/home/user/db2/mongo2/log/mongodb2.log
directoryperdb=true
logappend=true
replSet=repset
bind_ip=0.0.0.0
port=27018
oplogSize=1024
fork=true
noprealloc=true
#keyFile=/home/user/db2/mongo2/data/keyfile
#auth=true
wiredTigerCacheSizeGB = 12


--启动两个mongo实例
/home/user/db1/mongo1/mongodb1/bin/mongod -f /home/user/db1/mongo1/mongodb1_jiqun.conf
/home/user/db2/mongo2/mongodb2/bin/mongod -f /home/user/db2/mongo2/mongodb2_jiqun.conf
```



### 配置集群信息

三台mongo全部启动后，使用本地mongoshell连入任意节点配置如下信息。

```
config = {
    "_id" : "repset",
    "members" : [
    {
    "_id" : 0,
    "host" : "192.168.1.241:27017"
    },
    {
    "_id" : 1,
    "host" : "192.168.1.242:27017"
    },
    {
    "_id" : 2,
    "host" : "192.168.1.242:27018"
    }
    ]
}



rs.initiate(conf);
```

至此，两台服务器的三节点mongo数据库集群搭建完毕！！！



### mongo常用命令

```
--查看集群状态
rs.status();

--副本集可查询（相对会话）
db.getMongo().setSlaveOk();

--增加节点
rs.add({"host":"127.0.0.1:1234"});

--增加投票节点
rs.addArb()

--查看配置信息
db.serverCmdLineOpts()

--强制重新配置集群
rs.reconfig(config,{"force":true})

--获取配置集信息
db.getReplicationInfo()
```