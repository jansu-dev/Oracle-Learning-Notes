---
title:[Mongo]--Java因安全策略无法连接3版本Mongo
date:2020-09-03
---

​		

​		Java开发反应mongo数据库换版本之后，存在无法连接的状况，经过搜索，最终定位并解决了这次因版本认证策略导致的无法连接问题。

### 问题原因

**特别注意：**

1. mongodb认证机制有2种：SCRAM-SHA-1 和 MONGODB-CR。3.0之后版本默认为：SCRAM-SHA-1；值为3表示：MONGODB-CR、值为5表示：SCRAM-SHA-1
2. spring-mongodb默认为：MONGODB-CR，且不支持设置认证方式；



### 解决方法

在Mongo数据库中，使用命令修改mongodb的认证方式。

1、查看auth认证方式

```
use admin
db.system.version.find({"_id":"authSchema"})
```

2、删除之前设置的所有用户

```
db.system.users.remove({})
```

3、删除原auth认证方式，并设置为MONGODB-CR

```
db.system.version.remove({"_id":"authSchema"})
db.system.version.insert({"_id":"authSchema","currentVersion":3})
```

4、重新添加admin用户（超级管理员）

```
use admin
db.createUser({user:"admin",pwd:"123456",roles:[{role:"userAdminAnyDatabase",db:"admin"}]})
```



### 参考文章

- [csdn-后端老A的文章](https://blog.csdn.net/sanpangouba/article/details/78953556)