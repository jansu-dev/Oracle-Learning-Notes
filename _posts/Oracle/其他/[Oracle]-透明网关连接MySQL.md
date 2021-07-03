---
title:[oracle]-透明网关连接MySQL
date:2020-12-09
---



linux环境下，Oracle数据库通过DBLink远程连接MySQL数据库。

- [ODBC概念讲解](ODBC概念讲解)
- [ODBC资源下载](ODBC资源下载)

- [配置UnixODBC](配置UnixODBC)

- [配置odbc连接MyDQL](配置odbc连接MyDQL)

- [配置监听](配置监听)

- [配置db_link](配置db_link)

- [可能会遇到的问题](可能会遇到的问题)
- [参考文章](参考文章)





### ODBC概念讲解

##### 实验环境信息

1、Oracle
①操作系统：Linux X86-64
②数据库版本：11.2.0.4.0
③字符集：SIMPLIFIED CHINESE_CHINA.AL32UTF8

2、MySQL
①操作系统：Linux i686
②数据库版本：5.7.21
③字符集：UTF8

### ODBC资源下载

> [***MySQL官方资源地址\***](https://dev.mysql.com/downloads/connector/odbc/)

> [***JanNest共享资源地址\***](https://dev.mysql.com/downloads/connector/odbc/)



### 配置UnixODBC

1、下载UnixODBC安装包

```
软件地址：ftp://ftp.unixodbc.org/pub/unixODBC/unixODBC-2.3.0.tar.gz
```

2、安装

①root用户执行

```
cd /usr/local
    tar zxvf unixODBC-2.3.0.tar.gz
```

② 编译安装

```
cd unixODBC-2.3.0/
    ./configure --prefix=/usr/local/unixODBC-2.3.0 --includedir=/usr/include --libdir=/usr/lib64 --bindir=/usr/bin --sysconfdir=/etc

    make
    make install
```

3、测试
执行命令：odbcinst -j，如果安装成功会显示：

```
[root@vbox66 local]# odbcinst -j
unixODBC 2.3.0
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /root/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
[root@vbox66 local]# 
unixODBC所需的头文件都被安装到了/usr/inlucde下，编译好的库文件安装到了/usr/lib64下，与unixODBC相关的可执行文件安装到了/usr/bin下，配置文件放到了/etc下。
```



### 配置odbc连接MyDQL

1、下载文件

```
https://cdn.mysql.com//Downloads/Connector-ODBC/5.1/mysql-connector-odbc-5.1.13-1.x86_64.rpm
```

2、rpm安装

```
rpm -ivh mysql-connector-odbc-5.1.13-1.x86_64.rpm
```

3、配置odbc.ini文件

```
vi /etc/odbc.ini
[testdb]
Description = mysql
Driver = MySQL ODBC 5.1 Driver
Server = 192.169.31.103            //MySQL服务器IP
Database = test                   //MySQL数据库名
Port = 3306                      //端口
USER = user_name                //数据库用户名
Password = passwd                //用户民密码
Socket =
Option = 3
Stmt =
CHARSET = UTF8                  //数据库字符集
```

4、测试odbc连接MySQL
运行命令： isql testdb -v //testdb为odbc.ini文件中中括号中的内容

```
[root@vbox66 lib64]# isql testdb -v
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
|                                       |
+---------------------------------------+
SQL> show databases;
+-----------------------------------------------------------------+
| Database                                                        |
+-----------------------------------------------------------------+
| information_schema                                              |
| aaa                                                             |
| jrjctest                                                        |
| mysql                                                           |
| otter                                                           |
| performance_schema                                              |
| sbtest                                                          |
| sys                                                             |
| test                                                            |
| test1                                                           |
| test_wr                                                         |
| wr                                                              |
| wr_test1                                                        |
| www                                                             |
+-----------------------------------------------------------------+
SQLRowCount returns 14
14 rows fetched
```

odbc连接MySQL成功！



### 配置监听

1、修改环境变量

```
su - oracle
vi .bash_profile
export ODBCINI=/etc/odbc.ini
2、配置Oracle监听
①cd $ORACLE_HOME/network/admin
vi listener.ora
添加如下信息：
SID_LIST_LISTENER=
  (SID_LIST=
      (SID_DESC=
         (SID_NAME=dg4odbc)
         (ORACLE_HOME=/u01/app/oracle/product/11.2.0/db_1)
         (PROGRAM=dg4odbc)
      )
  )
```

官方文档解释如下：

②修改tnsnames.ora

```
vi tnsnames.ora
添加如下信息：
dg4odbc =
  (DESCRIPTION=
    (ADDRESS = (PROTOCOL =tcp)(HOST = vbox66)(PORT = 1521))
    (CONNECT_DATA =
      (SID = dg4odbc))
      (HS=OK)
  )
官方文档解释如下：
image
image
```

③配置ODBC监听

```
cd $ORACLE_HOME/hs/admin
vi initdg4odbc.ora  //注意，这里init后面的内容要和之前配置 SID_NAME（dg4odbc）一致
HS_FDS_CONNECT_INFO=testdb
HS_FDS_TRACE_LEVEL = on
HS_FDS_SHAREABLE_NAME=/usr/lib64/libodbc.so
HS_FDS_SUPPORT_STATISTICS=FALSE
HS_LANGUAGE="simplified chinese_china.al32utf8"    //提供具有非oracle数据源的字符集、语言和区域信息的异构服务
HS_NLS_NCHAR=UCS2     //NVARCHAR/NCHAR和图形数据类型通常以Unicode格式存储数据。unicode字符集因数据库的不同而不同,设置此参数外部数据库保持一致

set ODBCINI=/etc/odbc.ini
```

④重启测试监听配置

```
oracle用户执行：

lsnrctl stop
lsnrctl start
alter system register;(数据库中执行)
lsnrctl status;
image

tnsping dg4odbc
image
```

显示以上信息则表示监听配置成功!



### 配置db_link

3、创建DBLink

```
①su - oracle
sqlplus / as sysdba
create public database link myodbc connect to "root" identified by "123123" using 'dg4odbc'; --注意：wangrui和wangruit是MySQL用户名和密码，都需要使用双引号，dg4odbc使用单引号。
②select * from "t1"@myodbc;
image
```



### 可能会遇到的问题

1、

```
SYS@vbox66in>select "id" from t1@myodbc1;
select "id" from t1@myodbc1
*
第 1 行出现错误:
ORA-28500: 连接 ORACLE 到非 Oracle 系统时返回此信息:
[
```

原因：有一些内容显示不全，检查odbc监听文件，查看HS_LANGUAGE参数，配置为和数据库字符集一致

2、

```
SYS@vbox66in>select "id" from "t1"@myodbc1;
select "id" from "t1"@myodbc1
*
```

第 1 行出现错误:
ORA-28500: 连接 ORACLE 到非 Oracle 系统时返回此信息: ORA-28541:
HS 初始化文件的第 8 行发生错误。 ORA-02063:
紧接着 2 lines (起自 MYODBC1)
原因：NVARCHAR/NCHAR和图形数据类型通常以Unicode格式存储数据。unicode字符集因数据库的不同而不同，需要修改HS_NLS_NCHAR=UCS2
image

3、

```
SYS@vbox66in>select * from a2@myodbc1;
select * from a2@myodbc1
*
第 1 行出现错误:
ORA-00942: 表或视图不存在
[MySQL][ODBC 5.1 Driver][mysqld-5.7.21-log]Table 'test.A2' doesn't exist
{42S02,NativeErr = 1146}
ORA-02063: 紧接着 2 lines (起自 MYODBC1)
```

原因：MySQL数据中是区分大小写的，表需要用双引号引起来

### 参考文章

- [参考文章1-weixin_34026276的文章](https://blog.csdn.net/weixin_34026276/article/details/89559311)