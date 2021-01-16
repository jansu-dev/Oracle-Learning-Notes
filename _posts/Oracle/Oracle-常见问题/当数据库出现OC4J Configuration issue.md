---
title: 当数据库出现OC4J Configuration issue
date: 2019-9-1
categories: Oracle
tags: Oracle常见问题
---



##### OC4J Configuration issue. /u01/oracle/oc4j/j2ee/OC4J_DBConsole_bjcuug_prod not found. 

##### 开启监听:

lsnrctl start
sqlplus / as sysdba
alter system register;

emca -repos recreate
emctl -config dbcontrol db

如果一台主机存在多个数据库时，必须预先配置环境变量$ORACLE_SID

##### 常用commend：

​	emca -repos create            创建一个EM资料库
​	emca -repos recreate          重建一个EM资料库
​	emca -repos drop              删除一个EM资料库
​	emca -config dbcontrol db     配置数据库的 Database Control
​	emca -deconfig dbcontrol db   删除数据库的 Database Control配置
​	emca -reconfig ports          重新配置db control和agent的端口
​	emctl start console           关闭/启动EM
查看$ORACLE_HOME/install/portlist.ini 文件可以知道当前dbcontrol正在使用的端口
默认dbcontrol http端口1158，agent端口3938
如果要重新配置端口commend：
​	emca -reconfig ports -dbcontrol_http_port 1159
 	emca -reconfig ports -agent_port 3939