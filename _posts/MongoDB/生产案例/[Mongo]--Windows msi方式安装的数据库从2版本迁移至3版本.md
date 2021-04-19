---
title:[Mongo]--Windows msi方式安装的数据库从2版本迁移至3版本
date:2020-09-07
---



##### [Mongo]--Windows msi方式安装的数据库从2版本迁移至3版本

- [镜像的获取](镜像的获取)
- [安装MongoDB 2版本软件](安装MongoDB 2版本软件)
- [安装MongoDB 3版本软件](安装MongoDB 3版本软件)
- [创建MongoDB 2版本数据库](创建MongoDB 2版本数据库)
- [创建MongoDB 3版本数据库](创建MongoDB 3版本数据库)
- [mongodump跨版本迁移数据](mongodump跨版本迁移数据)
- [数据迁移](数据迁移)



### 镜像的获取

> **MongoDB 2.6.1 msi镜像：**
> [JanNest下载源](https://pan.baidu.com/s/121LSz-imX_JNag0vv6XHRQ)
> 提取码：2qj7
> **MongoDB 3.6.19 msi镜像**
> [JanNest下载源](https://pan.baidu.com/s/1YTrlW4oumjLbvkXC5Eg7Tw)
> 提取码：4mpo
> [官网下载源](https://www.mongodb.com/try/download/community)



### 安装MongoDB 2版本软件

图片为后期补充，C盘在后面代码中是E盘

![image.png](http://cdn.lifemini.cn/dbblog/20200916/d0a52e978a624050bb735055561b1091.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200916/a58d4dffc02042e59a55f524d42f6de3.png)
**将图片MongoDB3.6改为MongoDB2.6，本例安装在了与名称不对应的目录。**
**后面已手动改正，但图片未作修改！！！**



### 安装MongoDB 3版本软件

图片为后期补充，C盘在后面代码中是E盘

![image.png](http://cdn.lifemini.cn/dbblog/20200916/a6a882f6af8742e7a1052ef7b4ab234c.png)

修改下载路径
![image.png](http://cdn.lifemini.cn/dbblog/20200916/8c557ca4accc4769a49c56175fe6f397.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200916/1b3afc608dd44114b76773d79806f28f.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200916/58ceeda9ab464f4aaf65ba8acbe8f376.png)
**将Compass取消勾选，否则后面可能长期无法安装完成！**



### 创建MongoDB 2版本数据库

安装数据库，配置日志路径、数据文件路径；

```
C:\Users\Administrator>E:\mongo2.6\bin\mongod.exe --logpath "E:\mongo2.6\log\mon
go.log" --logappend --dbpath "E:\mongo2.6\data" --directoryperdb --install

2020-07-30T22:37:06.030+0800
2020-07-30T22:37:06.046+0800 warning: 32-bit servers don't have journaling enabl
ed by default. Please use --journal if you want durability.
2020-07-30T22:37:06.046+0800
```

启动数据库服务；

```
C:\Users\Administrator>net start MongoDB

MongoDB 服务已经启动成功。
```

使用Mongo客户端连接数据库；

```
C:\Users\Administrator>E:\mongo2.6\bin\mongo.exe
MongoDB shell version: 2.6.1
connecting to: test
Server has startup warnings:
2020-07-30T22:37:27.108+0800 [initandlisten]
2020-07-30T22:37:27.108+0800 [initandlisten] ** NOTE: This is a 32 bit MongoDB b
inary.
2020-07-30T22:37:27.108+0800 [initandlisten] **       32 bit builds are limited
to less than 2GB of data (or less with --journal).
2020-07-30T22:37:27.108+0800 [initandlisten] **       Note that journaling defau
lts to off for 32 bit and is currently off.
2020-07-30T22:37:27.108+0800 [initandlisten] **       See http://dochub.mongodb.
org/core/32bit
2020-07-30T22:37:27.108+0800 [initandlisten]
> show dbs;
admin  (empty)
local  0.078GB
szp    0.078GB


> use jannest
switched to db jannest


> db.jan.insert()
{ "name" : "mongo_test_20200907" }


> db.jan.find()
{ "_id" : ObjectId("5f22da327a85f3f5917b9020"), "name" : "mongo_test_20200907" }
```

### 创建MongoDB 3版本数据库

再E盘下创建conf目录，并添加配置文件

```
systemLog:
    destination: file
    path: E:\mongo3.6\log\mongod.log
storage:
    dbPath: E:\mongo3.6\data\db
```

创建数据库

```
C:\Users\Administrator>E:\mongo3.6\bin\mongod.exe --config "E:\mongo3.6\conf\mon
god.cfg" --install
```

将数据库实例加入Windows注册表，以服务的形式存在。

```
C:\Users\Administrator>sc.exe create MongoDB binPath= "E:\mongo3.6\bin\mongod.ex
e --service --config=\"E:\mongo3.6\conf\mongod.cfg\"" DisplayName= "MongoDB3.6"
start= "auto"
[SC] CreateService 失败 1073:
```

**如果版本升级发生在同一服务器，两哥版本数据库服务不可重名，否则会报错已存在。**

成功加载服务

```
C:\Users\Administrator>sc.exe create MongoDB3.6 binPath= "E:\mongo3.6\bin\mongod
.exe --service --config=\"E:\mongo3.6\conf\mongod.cfg\"" DisplayName= "MongoDB3.
6" start= "auto"
[SC] CreateService 成功
```

![image.png](http://cdn.lifemini.cn/dbblog/20200916/a9f81778af62401089b0797502513311.png)

启动服务

```
C:\Users\Administrator>net start MongoDB3.6
MongoDB3.6 服务正在启动 ..
MongoDB3.6 服务已经启动成功。
```

连接3版本数据库

```
C:\Users\Administrator>E:\mongo3.6\bin\mongo.exe
MongoDB shell version v3.6.19
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("de379024-b1cc-451c-a7e4-df7d56982cbc")
}
MongoDB server version: 3.6.19
Server has startup warnings:
2020-09-07T01:51:44.833+0800 I CONTROL  [initandlisten]
2020-09-07T01:51:44.833+0800 I CONTROL  [initandlisten] ** WARNING: Access contr
ol is not enabled for the database.
2020-09-07T01:51:44.833+0800 I CONTROL  [initandlisten] **          Read and wri
te access to data and configuration is unrestricted.
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten]
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten] ** WARNING: This server
is bound to localhost.
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten] **          Remote syste
ms will be unable to connect to this server.
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten] **          Start the se
rver with --bind_ip <address> to specify which IP
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten] **          addresses it
 should serve responses from, or with --bind_ip_all to
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten] **          bind to all
interfaces. If this behavior is desired, start the
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten] **          server with
--bind_ip 127.0.0.1 to disable this warning.
2020-09-07T01:51:44.834+0800 I CONTROL  [initandlisten]


> show dbs;
admin   0.078GB
config  0.078GB
local   0.078GB
>
```



### mongodump跨版本迁移数据

关闭数据库。

```
> use admin
switched to db admin

> db.shutdownServer()
2020-07-30T22:38:18.122+0800 DBClientCursor::init call() failed
server should be down...
2020-07-30T22:38:18.137+0800 trying reconnect to 127.0.0.1:27017 (127.0.0.1) fai
led
2020-07-30T22:38:19.153+0800 warning: Failed to connect to 127.0.0.1:27017, reas
on: errno:10061 由于目标计算机积极拒绝，无法连接。
2020-07-30T22:38:19.153+0800 reconnect 127.0.0.1:27017 (127.0.0.1) failed failed
 couldn't connect to server 127.0.0.1:27017 (127.0.0.1), connection attempt fail
ed
> exit
bye
```



### 数据迁移

创建测试用户数据；

```
C:\Users\Administrator>net start MongoDB
MongoDB 服务已经启动成功。


C:\Users\Administrator>E:\mongo2.6\bin\mongo.exe


> show dbs;
admin  (empty)
local  0.078GB
szp    0.078GB


> use szp;
switched to db szp


> show users;


> db.createUser({user:'Jan',pwd:'123123',roles:[{role:'readWrite',db:'jannest'}]})
Successfully added user: {
        "user" : "Jan",
        "roles" : [
                {
                        "role" : "readWrite",
                        "db" : "jannest"
                }
        ]
}


> show users;
{
        "_id" : "jannest.Jan",
        "user" : "Jan",
        "db" : "jannest",
        "roles" : [
                {
                        "role" : "readWrite",
                        "db" : "jannest"
                }
        ]
}


> use admin
switched to db admin


> exit
bye
```

2版本 MongoDB 端导出数据

```
C:\Users\Administrator>E:\mongo2.6\bin\mongodump.exe -u "root" -p "123123" --aut
henticationDatabase "admin" -o E:\mongo2.6\dump
connected to: 127.0.0.1
2020-09-07T02:03:05.960+0800 all dbs
2020-09-07T02:03:05.960+0800 DATABASE: admin     to     E:\mongo2.6\dump\admin
2020-09-07T02:03:05.960+0800    admin.system.indexes to E:\mongo2.6\dump\admin\s
ystem.indexes.bson
2020-09-07T02:03:05.960+0800             3 documents
2020-09-07T02:03:05.960+0800    admin.system.version to E:\mongo2.6\dump\admin\s
ystem.version.bson
2020-09-07T02:03:05.960+0800             1 documents
2020-09-07T02:03:05.960+0800    Metadata for admin.system.version to E:\mongo2.6
\dump\admin\system.version.metadata.json
2020-09-07T02:03:05.960+0800    admin.system.users to E:\mongo2.6\dump\admin\sys
tem.users.bson
2020-09-07T02:03:05.960+0800             2 documents
2020-09-07T02:03:05.960+0800    Metadata for admin.system.users to E:\mongo2.6\d
ump\admin\system.users.metadata.json
2020-09-07T02:03:05.960+0800 DATABASE: szp       to     E:\mongo2.6\dump\szp
2020-09-07T02:03:05.960+0800    szp.system.indexes to E:\mongo2.6\dump\szp\syste
m.indexes.bson
2020-09-07T02:03:05.960+0800             1 documents
2020-09-07T02:03:05.960+0800    szp.szp to E:\mongo2.6\dump\szp\szp.bson
2020-09-07T02:03:05.992+0800             1 documents
2020-09-07T02:03:05.992+0800    Metadata for szp.szp to E:\mongo2.6\dump\szp\szp
.metadata.json

C:\Users\Administrator>
```

3版本 MongoDB 端创建必要root用户；

```
C:\Users\Administrator>E:\mongo3.6\bin\mongo.exe
MongoDB shell version v3.6.19
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("97d16f36-21c5-4bc1-b138-2eba7b504277")
}
MongoDB server version: 3.6.19
Server has startup warnings:
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten]
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] ** WARNING: Access contr
ol is not enabled for the database.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          Read and wri
te access to data and configuration is unrestricted.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten]
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] ** WARNING: This server
is bound to localhost.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          Remote syste
ms will be unable to connect to this server.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          Start the se
rver with --bind_ip <address> to specify which IP
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          addresses it
 should serve responses from, or with --bind_ip_all to
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          bind to all
interfaces. If this behavior is desired, start the
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          server with
--bind_ip 127.0.0.1 to disable this warning.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten]


> use admin
switched to db admin


> db.createUser(
...   {
...    user: "root",
...    pwd: "123123",
...    roles:
...    [ "root"]
...   }
...  )
Successfully added user: { "user" : "root", "roles" : [ "root" ] }


> show users;
{
        "_id" : "admin.root",
        "userId" : UUID("c6b52cb2-db9c-4459-b97b-61b1c18a6a9b"),
        "user" : "root",
        "db" : "admin",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}


> exit
bye
```

3版本 MongoDB 端导入数据；

```
C:\Users\Administrator>E:\mongo3.6\bin\mongorestore.exe -u "root" -p "123123" --
authenticationDatabase "admin" E:\mongo2.6\dump
2020-09-07T02:09:08.866+0800    preparing collections to restore from
2020-09-07T02:09:08.889+0800    reading metadata for szp.szp from E:\mongo2.6\du
mp\szp\szp.metadata.json
2020-09-07T02:09:09.244+0800    restoring szp.szp from E:\mongo2.6\dump\szp\szp.
bson
2020-09-07T02:09:09.426+0800    no indexes to restore
2020-09-07T02:09:09.435+0800    finished restoring szp.szp (1 document)
2020-09-07T02:09:09.436+0800    restoring users from E:\mongo2.6\dump\admin\syst
em.users.bson
2020-09-07T02:09:10.595+0800    done

C:\Users\Administrator>E:\mongo3.6\bin\mongo.exe
MongoDB shell version v3.6.19
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("176f949b-fe62-416c-b704-edf6387f65d8")
}
MongoDB server version: 3.6.19
Server has startup warnings:
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten]
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] ** WARNING: Access contr
ol is not enabled for the database.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          Read and wri
te access to data and configuration is unrestricted.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten]
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] ** WARNING: This server
is bound to localhost.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          Remote syste
ms will be unable to connect to this server.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          Start the se
rver with --bind_ip <address> to specify which IP
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          addresses it
 should serve responses from, or with --bind_ip_all to
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          bind to all
interfaces. If this behavior is desired, start the
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten] **          server with
--bind_ip 127.0.0.1 to disable this warning.
2020-09-07T02:05:23.148+0800 I CONTROL  [initandlisten]
> show dbs;
admin   0.078GB
config  0.078GB
local   0.078GB
szp     0.078GB
> use szp
switched to db szp
> db.szp.find()
{ "_id" : ObjectId("5f22da327a85f3f5917b9020"), "name" : "mongo_test_20200907" }

> show tables;
system.indexes
szp
> show users;
{
        "_id" : "szp.Jan",
        "user" : "Jan",
        "db" : "szp",
        "roles" : [
                {
                        "role" : "readWrite",
                        "db" : "szp"
                }
        ]
}
>
```