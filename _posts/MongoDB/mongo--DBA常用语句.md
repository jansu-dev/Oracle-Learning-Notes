---
title:mongo--DBA常用语句
date:2020-07-20
---



##### mongo常用语句

- [用户管理](用户管理)
- [导入导出](导入导出)
- [备份恢复](备份恢复)
- [集群管理](集群管理)
- [碎片收缩](碎片收缩)
- [日志信息](日志信息)
- [参考文章](参考文章)





### 用户管理

角色解读

```
Built-In Roles（内置角色）    
1. 数据库用户角色：read、readWrite;
2. 数据库管理角色：dbAdmin、dbOwner、userAdmin；
3. 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
4. 备份恢复角色：backup、restore；
5. 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
6. 超级用户角色：root  
7. 其他角色dbOwner 、userAdmin、userAdminAnyDatabase（这里还有几个角色间接或直接提供了系统超级用户的访问）
7. 内部角色：__system    



Read：允许用户读取指定数据库
readWrite：允许用户读写指定数据库
dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
root：只在admin数据库中可用。超级账号，超级权限
```

root用户在sinopec6000数据库下
创建hollycrm用户
授予readWrite(读写角色)

```
use sinopec6000;
db.createUser({user:'hollycrm',pwd:'Mongo@passw0rd',roles:[{role:'readWrite',db:'sinopec6000'}]})
```

root用户在admin数据库下
先创建hollycrm用户
后授予readWriteAnyDatabase角色
后授予backup(备份角色)

```
use admin;
db.createUser({user:'hollycrm',pwd:'Mongo@passw0rd',roles:[{role:'backup',db:'admin'}]})
db.grantRolesToUser('hollycrm',[{role:'readWriteAnyDatabase',db:'admin'}])
db.grantRolesToUser('hollycrm',[{role:'backup',db:'admin'}])
```

回收hollycr用户的readWriteAnyDatabase(读写任何数据库角色)

```
db.revokeRolesFromUser('hollycrm',[{role:'readWriteAnyDatabase',db:'admin'}])
```

删除用户 hollycrm

```
db.dropUser('hollycrm')
```

修改密码 hollycrm

```
db.changeUserPassword('tank2','test');
```

#### 增删改查基本命令

首先查看集合中的数据

```
db.bike_bak.find()
```

查看所有数据库存储大小

```
var collectionNames= db.getCollectionNames();  
for (var i = 0; i < collectionNames.length; i++) {     
  var coll = db.getCollection(collectionNames[i]);   
  var stats = coll.stats();   
  print(stats.ns, stats.storageSize/1024/1024);  
}
```

删除集合中的数据

```
db.bike_bak.remove({"bikeId" : "pgdAVg"})
```

模糊查询

```
use test

db.getCollection("subscribe_test").find
(
  {$or:[
    {"name":{"$regex":"正大"}},
        {"content":{"$regex":"正大"}}
       ],
       "update_time":{$gte:1,$lte:2000000000},
       info_type:"00"
  }
)
```



### 导入导出

#### mongodump

-f 字段以逗号分隔
--type 导出的数据格式
--query 过滤的语句
--limit 导出条数

```
mongodump -h 127.0.0.1:27017 -d hollyreport -o /backup

mongorestore -h 127.0.0.1:27017 -d hollyreport /backup/corefuntion.bson
```

#### mongoexport

```
mongoexport -h 127.0.0.1:27017 -d mapdb -c bike -f bikeId,lat --type=json -o bike.csv --query='{"source":"ofo"}' --limit=1

mongoimport -d paper -c newslists --type json --file C:\Users\dell\Desktop\newslist.json
```



### 备份恢复

#### 备份

备份单个表

```
mongodump --authenticationDatabase admin -d myTest -c d -o /backup/mongodb/myTest_d_bak_201507021701.bak
```

备份单个库

```
mongodump --port 27017  --authenticationDatabase admin -d myTest -o  /backup/mongodb/
```

备份所有库

备份所有库(推荐使用添加--oplog参数)，
基于某一时间点的快照，只能用于备份全部库时才可用，单库和单表不适用。
同时，恢复时也要加上--oplogReplay参数，具体命令如下(下面是恢复单库的命令)：

```
mongodump  --authenticationDatabase admin  --port 27017 -o /root/bak 

mongodump -h 127.0.0.1 --port 27017   --oplog -o  /root/bak 

mongorestore  -d swrd --oplogReplay  /home/mongo/swrdbak/swrd/
```

#### 恢复

恢复单个库：

```
mongorestore  -u  superuser -p 123456 --port 27017  --authenticationDatabase admin -d myTest   /backup/mongodb/
```

恢复所有库

```
mongorestore   -u  superuser -p 123456 --port 27017  --authenticationDatabase admin  /root/bak
```

恢复单表

```
mongorestore -u  superuser -p 123456  --authenticationDatabase admin -d myTest -c d /backup/mongodb/myTest_d_bak_201507021701.bak/myTest/d.bson
```

导入导出实验

```
首先查看集合中的数据
db.bike_bak.find()

删除集合中的数据
db.bike_bak.remove({"bikeId" : "pgdAVg"})

查看集合是否包含数据
db.bike_bak.find()

导入数据
mongoimport --port 27030 -u sa -p Expressin@0618 -d mapdb -c bike_bak  --type=json --file bike.json

检查数据是否导入成功
db.bike_bak.find()
```



### 集群管理

```
# 使从库可见
db.getMongo().setSlaveOk();
```



### 碎片收缩

```
use YOURDB

db.runCommand({repairDatabase:1})
```



### 日志信息

Profile 信息内容详解：

　　ts-该命令在何时执行.

　　millis Time-该命令执行耗时，以毫秒记.

　　info-本命令的详细信息.

　　query-表明这是一个query查询操作.

　　ntoreturn-本次查询客户端要求返回的记录数.比如, findOne()命令执行时 ntoreturn 为 1.有limit(n) 条件时ntoreturn为n.

　　query-具体的查询条件(如x>3).

　　nscanned-本次查询扫描的记录数.

　　reslen-返回结果集的大小.

　　nreturned-本次查询实际返回的结果集.

　　update-表明这是一个update更新操作.

字段说明：
cursor：返回游标类型
isMultiKey：是否使用组合索引
n：返回文档数量
nscannedObjects：被扫描的文档数量
nscanned：被检查的文档或索引条目数量
scanAndOrder：是否在内存中排序
indexOnly：
nYields：该查询为了等待写操作执行等待的读锁的次数
nChunkSkips：
millis：耗时(毫秒)
indexBounds：所使用的索引
server： 服务器主机名
可以结合hint强制使用索引来分析。

二、开启profiling功能，设置日志级别，对日志进行分析
1.查看profiling级别：
db.getProfilingLevel()
2.设置profiling级别：
语法：db.setProfilingLevel(level,slowms)
level - profile的级别可以取0，1，2 表示的意义如下：

\#0 - 关闭性能分析，测试环境可以打开，生成环境关闭，对性能有很大影响;

\#1 - 开启慢查询日志，执行时间大于100毫秒的语句

\#2 - 开启所有操作日志
slowms - 慢查询时间阀值，默认100ms

```
> db.system.profile.find().sort({$natural:-1}).limit(1) 
{
 "ts": ISODate("2015-03-19T08:42:51.012Z"),
 "op": "query",
 "ns": "test.system.profile",
 "query": {
  "query": {

  },
  "orderby": {
   "$natural": -1
  }
 },
 "ntoreturn": 1,
 "ntoskip": 0,
 "nscanned": 1,
 "keyUpdates": 0,
 "numYield": 0,
 "lockStats": {
  "timeLockedMicros": {
   "r": NumberLong(118),
   "w": NumberLong(0)
  },
  "timeAcquiringMicros": {
   "r": NumberLong(6),
   "w": NumberLong(4)
  }
 },
 "nreturned": 1,
 "responseLength": 385,
 "millis": 0,
 "client": "127.0.0.1",
 "user": ""
}
```

参数介绍：
ts:操作执行时的时间戳
millis:执行操作所花的时间
query：数据库查询操作，查询字段信息包括ntoreturn,query,nscanned,reslen,nreturned
ntoreturn:从查询中返回客户端指定的对象数
query:查询操作信息
nscanned:在执行查询操作的时候扫描了多少对象
reslen:查询结果的大小
nreturned:从查询中返回的结果对象
update：数据库更新操作，
insert：数据库插入操作
getmore：大数据量查询

```
db.system.profile.find().pretty().limit(1) 
{ 
"ts" : ISODate("2015-03-19T08:36:13.451Z"), 
"op" : "command", 
"ns" : "test.$cmd", 
"command" : { 
"profile" : 2, 
"slowms" : 100 
}, 
"ntoreturn" : 1, 
"keyUpdates" : 0, 
"numYield" : 0, 
"lockStats" : { 
"timeLockedMicros" : { 
"r" : NumberLong(0), 
"w" : NumberLong(18) 
}, 
"timeAcquiringMicros" : { 
"r" : NumberLong(0), 
"w" : NumberLong(7) 
} 
}, 
"responseLength" : 58, 
"millis" : 0, 
"client" : "127.0.0.1", 
"user" : "" 
}
```

字段说明：
ts： 该命令在何时执行
op: 操作类型
query: 本命令的详细信息
responseLength: 返回结果集的大小
ntoreturn: 本次查询实际返回的结果集
millis: 该命令执行耗时，以毫秒记

1. 修改profiling大小

   ```
   > db.setProfilingLevel(0) 
   { "was" : 2, "slowms" : 100, "ok" : 1 } 
   > db.getProfilingLevel() 
   0 
   > db.system.profile.drop() 
   true 
   > db.createCollection("system.profile",{capped:true, size: 1000000}) 
   { "ok" : 1 } 
   > db.system.profile.stats() 
   { 
   "ns" : "test.system.profile", 
   "count" : 0, 
   "size" : 0, 
   "storageSize" : 1003520, 
   "numExtents" : 1, 
   "nindexes" : 0, 
   "lastExtentSize" : 1003520, 
   "paddingFactor" : 1, 
   "systemFlags" : 0, 
   "userFlags" : 0, 
   "totalIndexSize" : 0, 
   "indexSizes" : {
   ```

},
"capped" : true,
"max" : 2147483647,
"ok" : 1
}

```
**日志变量说明**

FSYNC：
– 强制mongod进程将所有挂起的写入从存储层刷新到磁盘.

keyUpdates： – 更新在操作中更改的索引键数.更改索引键的性能成本较低,因为数据库必须删除旧密钥并将新密钥插入B树索引.

numYield： – 允许其他操作完成的操作所产生的次数.通常,当操作需要访问MongoDB尚未完全读入内存的数据时,操作会产生.这允许在MongoDB读取数据以进行让步操作时,在内存中有数据的其他操作完成.

responseLength(reslen)： – 操作结果文档的长度(以字节为单位).较大的responseLength会影响性能.

nscanned： – MongoDB在索引中扫描以执行操作的文档数.通常,如果nscanned远高于nredurned,则数据库会扫描许多对象以查找目标对象.考虑创建一个索引来改善这一点.

nMatched： – 选择要更新的文档数.如果更新操作导致文档没有变化,例如$set expression将值更新为当前值,nMatched可以大于nModified.

nModified： – 更新现有文档的数量.如果更新/替换操作导致文档没有更改,例如将字段的值设置为其当前值,则nModified可能小于nMatched.

lockStats locks(micros)： – 操作获取和保持锁定所花费的时间(以微秒为单位).此字段报告以下锁定类型的数据：

R - global read lock
W - global write lock
r - database-specific read lock
w - database-specific write lock
timeLockedMicros： – 操作持有特定锁定的时间(以微秒为单位).对于需要多个锁的操作,例如锁定本地数据库以更新oplog的操作,此值可能长于操作的总长度(即millis).


### 引用文章
[博客园-H_Johnny的文章-角色讲解](https://www.cnblogs.com/dbabd/p/10811523.html#_label00)
```

### 

### 参考文章

- [博客园-H_Johnny的文章-角色讲解](https://www.cnblogs.com/dbabd/p/10811523.html#_label00)