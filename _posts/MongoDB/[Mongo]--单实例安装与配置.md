---
title:[Mongo]--单实例安装与配置
date:2020-06-28
---



##### MongoDB单实例配置文件

- [配置文件内容](配置文件内容)
- [用户创建](用户创建)
- [修改环境变量](修改环境变量)





### 配置文件内容

```
dbpath=/home/mongo4/data
logpath=/home/mongo4/logs/mongodb.log
directoryperdb=true
logappend=true
bind_ip=0.0.0.0
port=27017
fork=true
noprealloc=true
maxConns=200
auth=false
wiredTigerCacheSizeGB=2
```



### 用户创建

##### 创建root用户

```
use admin

db.createUser(
  {
    user: "root",
    pwd: "hollycrm",
    roles:
    [ "root"]
  }
)
```

##### 创建普通数据库用户

```
use monitor

db.createUser({user:'isnull',pwd:'isnull',roles:[{role:'readWrite',db:'monitor'}]})
```



### 修改环境变量

将最后三行注释掉，追加如下内容。

```
[root@localhost ~]# vi .bash_profile

#PATH=$PATH:$HOME/bin

#export PATH

MONGO_HOME=/usr/local/mongo/mongodb
PATH=$MONGO_HOME/bin:$PATH
export PATH
```