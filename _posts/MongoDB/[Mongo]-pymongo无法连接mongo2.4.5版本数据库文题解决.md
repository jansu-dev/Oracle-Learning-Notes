---
title:[Mongo]-pymongo无法连接mongo2.4.5版本数据库文题解决
date:2020-10-12
---



##### [Mongo]-pymongo无法连接mongo2。4.5版本数据库文题解决

https://stackoverflow.com/questions/48060354/configurationerror-server-at-127-0-0-127017-reports-wire-version-0-but-this-v

[关于MongoDB的URL连接时用户名或密码中出现特殊字符问题](https://blog.csdn.net/u013732444/article/details/78229177)

SCRAM-SHA-1与MONGODB-CR

安装的是pymongo， 服务端在树莓派，运行的是mongod的服务。
错误是：
raise ConfigurationError(self._incompatible_err)
pymongo.errors.ConfigurationError: Server at 192.168.0.100:27017 reports wire version 0, but this version of PyMongo requires at least 2 (MongoDB 2.6).

因为连接本机的mongd server，没有出现这个问题。 所以问题应该处在版本上。
然后把pymongo的版本降下去，原来是3.6的版本，然后降到3.2. 重试后问题就解决了。

sudo pip install pymongo==3.2

[pymongo连接树莓派的mongo server出现错误](https://blog.csdn.net/nicholas_dlut/article/details/80933844)

##### pymongo连接mongodb的三种认证方式

https://blog.csdn.net/hrainning/article/details/84980622