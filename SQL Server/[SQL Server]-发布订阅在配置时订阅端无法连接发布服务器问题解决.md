---
title:[SQL Server]-发布订阅在配置时订阅端无法连接发布服务器问题解决
date:2020-10-18
---



​		问题始于我所服务的客户SQL Server数据库因为磁盘空间有限，需要使用分离附加功能将原来数据文件迁移至新的大容量盘符下。因分离附加无法在配置发布订阅的前提下进行，所以在迁移前后需重配发布订阅，但是迁移后出现订阅段无法连接发布端服务器的情况。



### 问题描述

迁移数据文件后，数据库附加正常，但当我在订阅端配置时出现无法连接发布端服务器的情况。
![image.png](http://cdn.lifemini.cn/dbblog/20201018/8e9952afdc484376a2a4199a0fe2826b.png)

报错截图如下：

![image.png](http://cdn.lifemini.cn/dbblog/20201018/cc3d87eb3d204bfcbc207811ec2c44e9.png)



### 尝试方向

网上普遍解决方法

1. 默认实例名称和当前的实例名称不一致，修改@@servername方法解决。
2. 在连接发端服务器时，今天填写主机名，不加后面的 “\实例名”。

<span style='color:red'>注意：</span>

<span style='color:red'>1. 我的问题不属于第一种解决办法。</span>
<span style='color:red'>2. 在这台有问题的生产电脑上，会报错“SQL Server复制需要有......”无法使用这种方法，但是我在其他生产主机上测试这种方法也是可用的。</span>

![image.png](http://cdn.lifemini.cn/dbblog/20201018/093143a4bb19462ca2ac7db949cec508.png)



### 解决问题

首先，为了防止分发服务中存在干扰，重新配置分发服务。但直接删除使用图形化工具显示分发服务正在使用中无法删除。于是，使用如下命令查出响应线程，再kill掉该线程。

```
SELECT [Spid] = session_id , ecid ,
[Database] = DB_NAME(sp.dbid) ,[User] = nt_username ,
[Status] = er.status , [Wait] = wait_type ,
[Individual Query] = SUBSTRING(qt.text,
er.statement_start_offset / 2,
(CASE WHEN er.statement_end_offset = -1
THEN LEN(CONVERT(NVARCHAR(MAX), qt.text))* 2
ELSE er.statement_end_offset
END - er.statement_start_offset )/ 2) ,
[Parent Query] = qt.text , Program = program_name ,hostname , nt_domain , start_time
FROM sys.dm_exec_requests er
INNER JOIN sys.sysprocesses sp ON er.session_id = sp.spid
CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) AS qt
WHERE session_id > 50 -- Ignore system spids.
AND session_id NOT IN ( @@SPID ) -- Ignore this current statement.
ORDER BY 1 ,2;
```

但是，在这里我的操作顺序出现了事务，先删除了分发数据库。

```
use master;
go
exec sp_dropdistributiondb @database = N'我的分发数据库名称'
go
```

由于我才用的时先删除分发数据库，而没有删除分发服务字典信息，所以出现报错:“...若要将服务器配置为分发服务器，首先将原分发服务器删除”。

![image.png](http://cdn.lifemini.cn/dbblog/20201018/dc46eec1fcb248eaa9de4c57ed9317a4.png)

注意：使用如下命令解决上述问题

```
SELECT * FROM msdb.dbo.MSdistributor;


USE master;
GO
EXEC sp_dropdistributor;
GO

SELECT * FROM msdb.dbo.MSdistributor;
```

至此，重新配置分发服务完毕。

正式开始解决本次问题

在订阅端的SQL Server配置管理器中，创建发布端对应的别名。
![image.png](http://cdn.lifemini.cn/dbblog/20201018/4e885c628b4c4758a21eb9df92e733d8.png)

注意：一定要选择，32位的创建，配置64位的可能无法生效。

![image.png](http://cdn.lifemini.cn/dbblog/20201018/23ba54fa10af451ba0446de22202f166.png)

再次依照顺序创建订阅，使用 “机器名\实例名” 就可以正常连接发布端服务器了。



### 问题总结

SQL Server的别名有些类似于Oracle的tnsname服务别名配置，
本次问题的解决相当于骗过了发布订阅的一些校验（本人猜测），
利用别名机制直接连接到了发布端。



### 参考文章

- ##### [博客园-潇湘隐者的文章](https://www.cnblogs.com/kerrycode/p/4010809.html)

- ##### [csdn-weixin_34049948的文章](https://blog.csdn.net/weixin_34049948/article/details/85990311)

- ##### [csdn-mituan1234567的文章](https://blog.csdn.net/mituan1234567/article/details/8620582?utm_source=blogxgwz1)

- ##### [海阔天空的文章](https://www.cnblogs.com/bigsearenhai/p/4330125.html)