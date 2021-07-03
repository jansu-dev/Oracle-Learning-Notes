---
title:[Mongo]--物理冷备数据迁移
date:2020-06-16
---



##### 在Linux的OS层采用scp的方法，对数据迁移。

- [准备工作](准备工作)

- [开始迁移](开始迁移)

- [mongo单节点配置参数](mongo单节点配置参数)

  

### 准备工作

节点2

```
mkdir -p /home/db
```



### 开始迁移

节点1

```
use admin;

db.shutdownServer();

exit;

ps -ef|grep mongo

zip mongo1_data.zip ./data

scp mongo1_data.zip 192.168.1.241:/home/db/

scp mongodb.conf 192.168.1.241:/home/db/

scp mongodb-linux-x86_64-rhel62-4.2.6.tgz root@192.168.1.241:/home/
```

节点2

```
tar -xzvf /home/mongodb-linux-x86_64-rhel62-4.2.6.tgz -C /home/db

mv mongodb-linux-x86_64-rhel62-4.2.6 mongo4

cd mongo4/bin/

unzip /home/mongo1_data.zip -d /home/db

/home/db/mongo4/bin/mongod -f /home/db/mongo4/mongodb.conf
```

验证迁移

```
/home/db/mongo4/bin/mongo -u USER_NAME -p PASSWORD

show dbs;
```



### mongo单节点配置参数

```
dbpath=/home/db/data
logpath=/home/db/logs/mongodb.log
directoryperdb=true
logappend=true
bind_ip=0.0.0.0
port=27017
fork=true
#noprealloc=true
maxConns=200
auth=false
wiredTigerCacheSizeGB=2
```