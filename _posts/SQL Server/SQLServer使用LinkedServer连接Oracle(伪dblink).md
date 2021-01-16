#



SQL Server采用linked server建立数据库与数据库之间的连接，功能与Oracle的dblink相似。
连接Oracle需要<span style="color:red">OraOLEDB.oracle</span>接口，但sql server本身并不默认安装，需要手动安装oracle客户端。

![](http://cdn.lifemini.cn/dbblog/20210115/48d71c6b987f4dc39fd935f94dd6d1a1.png)




文章大致分为：

> * 安装Oracle客户端 
> * 建立linked server



### 1.安装Oracle客户端



![](http://cdn.lifemini.cn/dbblog/20210115/851f1aa773824b858c37d4c695d9057b.png)

##### 下载地址：

[Oracle官网下载地址：https://www.oracle.com/database/technologies/112010-win64soft.html](https://www.oracle.com/database/technologies/112010-win64soft.html)


打开压缩包找到setup.exe右键打开，或以管理员身份运行；


![](http://cdn.lifemini.cn/dbblog/20210115/ccf961985f2e45a1987c60d5c575a3ab.png)

可能会出现不满足要求的情况；
<div style="color:red">注意：可能需要等待很长时间才能弹出此弹窗，不要以为系统无响应或者下错软件了。</div>


![](http://cdn.lifemini.cn/dbblog/20210115/0871dcf285d04da9b300ba169dd5b046.png)


修改解压该路径下/client/stage/cvu下的两个xml文件便可以解决此问题；
```
cvu_prereq.xml
oracle.client_InstantClient.xml
```


![](http://cdn.lifemini.cn/dbblog/20210115/523e84132b2c4a8c9e6183ccb306e25a.png)


```
	 <OPERATING_SYSTEM RELEASE="6.2">
			<VERSION VALUE="3"/>
			<ARCHITECTURE VALUE="64-bit"/>
			<NAME VALUE="WindowsServer2012R2"/>
			<ENV_VAR_LIST>
					<ENV_VAR NAME="PATH" MAX_LENGTH="1023" />
			</ENV_VAR_LIST>
		</OPERATING_SYSTEM>
```

在两个xml文件的图片所示位置追加上述代码部分；

<div style="color:red">
注意：
</br>
1.RELEASE="6.2"而不是RELEASE="6.1"
</br>
2.VALUE="WindowsServer2012R2"写的值应该是系统的版本
</div>



![](http://cdn.lifemini.cn/dbblog/20210115/963854e8ea004e95bbd9a1d855de2522.png)

![](http://cdn.lifemini.cn/dbblog/20210115/40a4bfc0dbbb4a53b5ff6a81de503ae0.png)

选择管理员模式；

![](http://cdn.lifemini.cn/dbblog/20210115/08943d5d3fbb419782cb89cc4d402d67.png)

![](http://cdn.lifemini.cn/dbblog/20210115/53c23c580db840d19063f587a10134f3.png)

最好，自己建立一个文件夹，存放oracle客户端软件以便后期管理。
本例为：C:\oracle_client

![](http://cdn.lifemini.cn/dbblog/20210115/dec284e460794630bab1c46fae8ec4f4.png)

![](http://cdn.lifemini.cn/dbblog/20210115/8102e750744e450ab19f341a10a93e1c.png)


![](http://cdn.lifemini.cn/dbblog/20210115/a39024356db3486db2a3caf32667c8f9.png)

安装oracle client后，OraOLEDB.Oracle接口会被sql server自动识别。

![](http://cdn.lifemini.cn/dbblog/20210115/f810977fcf744322b9e529bdf7228048.png)



### 2.建立linked server

```
-- 建立连接服务器
EXEC sp_addlinkedserver
@server='CORESU_LINK',	--被访问的服务器别名
@srvproduct='',	--SqlServer默认不需要写，或ORACL
@provider='OraOLEDB.Oracle',	--数据库访问接口
@datasrc='192.168.1.200:1521/prod'	--要访问的数据库相关信息
GO

-- 映射登陆用户
EXEC sp_addlinkedsrvlogin
@rmtsrvname='CORESU_LINK',	--被访问的服务器别名
@useself='false',	--固定这么写
@rmtuser='scott',	--被访问的数据库用户名
@rmtpassword='scott'	--被访问的数据库用户密码
GO

-- 测试查询
select * from CORESU_LINK..SCOTT.EMP;

-- 删除连接服务器
EXEC sp_dropserver "CORESU_LINK"
```
<div style="color:red">
注意：连接服务器名、oracle用户名、表名均要大写。
</div>
![](http://cdn.lifemini.cn/dbblog/20210115/bdf6d358d1444e3388840c2db356e9e5.png)