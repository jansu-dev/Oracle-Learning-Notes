---
title:[Mongo]--跨版本升级mongodump迁移数据出现duplicate field问题解决方案
date:2020-09-09
---

​		

​		MongoDB不同版本之间存在数据不兼容现象，本例解决了duplicate field源数据冲突问题。

### 问题现象

**报错信息如下：**

```
2018-04-06T10:47:22.964+0200    finalizing intent manager with multi-database longest task first prioritizer
2018-04-06T10:47:22.964+0200    restoring up to 4 collections in parallel
2018-04-06T10:47:22.964+0200    starting restore routine with id=3
2018-04-06T10:47:22.964+0200    starting restore routine with id=0
2018-04-06T10:47:22.964+0200    starting restore routine with id=1
2018-04-06T10:47:22.964+0200    starting restore routine with id=2
2018-04-06T10:47:22.964+0200    reading metadata for DBNAME.evenements from /home/tmp/path/folder/evenements.metadata.json
2018-04-06T10:47:22.964+0200    creating collection DBNAME.evenements using options from metadata
2018-04-06T10:47:22.964+0200    using collection options: bson.D{bson.DocElem{Name:"create", Value:"evenements"}, bson.DocElem{Name:"idIndex", Value:mongorestore.IndexDocument{Options:bson.M{"name":"_id_", "ns":"DBNAME.evenements"}, Key:bson.D{bson.DocElem{Name:"_id", Value:1}}, PartialFilterExpression:bson.D(nil)}}}
2018-04-06T10:47:22.965+0200    Failed: DBNAME.evenements: error creating collection DBNAME.evenements: error running create command: BSON field 'OperationSessionInfo.create' is a duplicate field
```

关键报错信息：error running create command: BSON field 'OperationSessionInfo.create' is a duplicate field

说明跨版本迁移存在源数据上的冲突。



### 解决原理

在导入数据的时候，MongoDB自身会自动创建源数据类型，所以无需导入元数据。



### 解决办法

将mongo2版本dump出来的源数据文件全部删除。

![image.png](http://cdn.lifemini.cn/dbblog/20200916/013f8953e4854695aa0a312dd97f93e1.png)
**collection的dump文件中带有metadata关键子的元数据文件全部删除！！！**
之后便可在MongoDB 3版本数据库成功导入。