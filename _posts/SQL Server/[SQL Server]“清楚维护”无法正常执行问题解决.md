---
title:[SQL Server]-“清楚维护”无法正常执行问题解决
date:2020-10-26
---





#### [SQL Server]-“清楚维护”无法正常执行问题解决



### 错误描述

![image.png](http://cdn.lifemini.cn/dbblog/20201109/a53a6edde89b43d9b7fa6af65f1c3226.png)



### 解决方法

修改SQL SERVER启动帐户为SYSTEM，对数据库所在分区添加SYSTEM用户完全控制权限，问题解决

### 参考文章

> [**孤城浪子的文章**](http://www.gclz.cn/post/59/)