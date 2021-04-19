---
title:[oracle]-透明网关连接SQL server
date:2020-10-14
---



#### 目录

- ##### [透明网关](透明网关)

- ##### [透明网关软件下载](透明网关软件下载)

- ##### [安装gateway](安装gateway)

- ##### [配置透明网关层SID信息](配置透明网关层SID信息)

- ###### [配置透明网关层监听](配置透明网关层监听)

- ###### [Oracle服务器配置tns](Oracle服务器配置tns)

- ###### [建立DB link](建立DB link)

- ###### [遇到的错误](遇到的错误)

- ###### [参考文章](参考文章)





### 透明网关

### 透明网关软件下载

> ###### [Oracle官方资源地址](https://wenku.baidu.com/view/44afaa8884868762caaed5d2.html)

> ###### [JanNest共享资源地址](https://wenku.baidu.com/view/44afaa8884868762caaed5d2.html)



### 安装gateway

解压linux.x64_11gR2_gateways.zip；
运行setup安装即可；
这里我们将透明网关和SQLServer数据库安在了一台服务器上。

填写SQLServer数据库服务器主机名，如：172.17.22.230；数据库名称：HR

安装完Gateway软件后，在ORACLE_HOME目录(D:\product\11.2.0\tg_1)下有一下dg4msql的目录，
这就是Gateway软件的目录了。



### 配置透明网关层SID信息

指明要访问的MSSQL数据库
在D:\Oracle\product\11.2.0\tg_1\dg4msql\admin\initdg4msql.ora目录下有一个initdg4msql.ora的文件。
该文件是Gateway的初始参数文件，描述连接的是哪个SQL Server数据库。
该文件的格式是initSID.ora，这里的SID在后面需要用到，系统默认的是dg4msql，一般情况这样就可以了。
如果改名，如使用HR作为SID，则文件名变成initHR.ora。文件内容如下

```
HS_FDS_CONNECT_INFO=[172.17.22.230]//HR
HS_FDS_TRACE_LEVEL=OFF
HS_FDS_RECOVERY_ACCOUNT=RECOVER
HS_FDS_RECOVERY_PWD=RECOVER
```

只要修改HS_FDS_CONNECT_INFO参数就可以了。
格式是：[hostname:port]/serverinstance/databasename，
其中hostname是机器名称或IP，PORT是SQL Server的端口号，SQL Server2005默认为1433.
serverinstance是SQL Server的实例名，一般空着就行。Databasename是SQL Server的数据库名。
因为我们在安装过程中指定了主机名和数据库名，这里已经有信息了。



### 配置透明网关层监听

透明网关层的监听配置文件: D:\Oracle\product\11.2.0\tg_1\NETWORK\ADMIN\listener.ora

这里PROGRAM指定应用程序名称，因为实例配置文件在D:\Oracle\product\11.2.0\tg_1\dg4msql\admin\initdg4msql.ora，
所以PROGRAM不能改变, SID_NAME就是前面init.ora文件名里指定的SID

```
SID_LIST_LISTENER =
   (SID_LIST =
     (SID_DESC =
     (GLOBAL_DBNAME = HR)
     (PROGRAM = dg4msql)
     (SID_NAME = HR)
     (ORACLE_HOME = D:\product\11.2.0\tg_1)
    )
  )


LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 172.17.22.230)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

ADR_BASE_LISTENER = D:\product\11.2.0\tg_1
```

启动监听

```
C:\Users\Administrator>lsnrctl start

LSNRCTL for 64-bit Windows: Version 11.2.0.1.0 - Production on 20-7月 -2016 15:0
8:31

Copyright (c) 1991, 2010, Oracle.  All rights reserved.

启动tnslsnr: 请稍候...

TNSLSNR for 64-bit Windows: Version 11.2.0.1.0 - Production
系统参数文件为D:\product\11.2.0\tg_1\network\admin\listener.ora
写入d:\product\11.2.0\tg_1\diag\tnslsnr\WIN-158KFV5FSBR\listener\alert\log.xml的
日志信息
监听: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=172.17.22.230)(PORT=1521)))
监听: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(PIPENAME=\\.\pipe\EXTPROC1521ipc)))

正在连接到 (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=172.17.22.230)(PORT=1521)))

LISTENER 的 STATUS
------------------------
别名                      LISTENER
版本                      TNSLSNR for 64-bit Windows: Version 11.2.0.1.0 - Produ
ction
启动日期                  20-7月 -2016 15:08:47
正常运行时间              0 天 0 小时 0 分 3 秒
跟踪级别                  off
安全性                    ON: Local OS Authentication
SNMP                      OFF
监听程序参数文件          D:\product\11.2.0\tg_1\network\admin\listener.ora
监听程序日志文件          d:\product\11.2.0\tg_1\diag\tnslsnr\WIN-158KFV5FSBR\li
stener\alert\log.xml
监听端点概要...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=172.17.22.230)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(PIPENAME=\\.\pipe\EXTPROC1521ipc)))
服务摘要..
服务 "HR" 包含 1 个实例。
  实例 "HR", 状态 UNKNOWN, 包含此服务的 1 个处理程序...
命令执行成功

C:\Users\Administrator>
```

也可以去windows的服务中查看OracleOraGtw11g_home1TNSListener是否已经启动



### Oracle服务器配置tns

在需要建立dblink的Oracle数据库所在服务器，配置/u01/oracle/app/product/11.2.0/dbhome_1/network/admin/tnsnames.ora

```
dg4msql =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.11)(PORT = 1521))
    (CONNECT_DATA = (SID=dg4msql))
    (HS=OK)
  )
```

测试tns

```
[oracle@elearningtest admin]$ tnsping HR

TNS Ping Utility for Linux: Version 11.2.0.4.0 - Production on 20-JUL-2016 15:14:22

Copyright (c) 1997, 2013, Oracle.  All rights reserved.

Used parameter files:
/u01/oracle/app/product/11.2.0/dbhome_1/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST =172.17.22.230)(PORT = 1521)) (CONNECT_DATA=(SID=HR)) (HS=OK))
OK (10 msec)
```



### 建立DB link

```
create public database link ms connect to sa identified by "1qaz!QAZ" using 'Sales';
```

##### 测试

```
SQL> select 1 from dual@ms;

1

----------

1
```



### 遇到的错误

```
错误1:initdg4msql.ora错误
[SQL]Select 1 from dual@HRdbLink
[Err] ORA-28545: 连接代理时 Net8 诊断到错误
Unable to retrieve text of NETWORK/NCR message 65535
ORA-02063: 紧接着 2 lines (起自 HRDBLINK)
原因:

D:\Oracle\product\11.2.0\tg_1\dg4msql\admin\initdg4msql.ora的HS_FDS_CONNECT_INFO值配置错误
```

错误2.透明网关的监听卡住，无法重启
相关进程没有彻底关闭，杀掉相关进程 如dg4msql.exe

```
create public database link ms connect to test identified by test using 'dg4msql';

drop public database link ms;


select * from a@ms;
```



### 参考文章

- ###### [百度文库Oracle 异构服务](https://wenku.baidu.com/view/44afaa8884868762caaed5d2.html)


- ###### [小强斋太-Study Notes的文章](https://www.cnblogs.com/xqzt/p/5688659.html)