---
title:[Oracle]--11g数据库静默安装与部署
date:2020-09-03
---





##### [Oracle]--11g数据库静默安装与部署

```
[oracle@sgdSSD-32 response]$ vi /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.2.196.75 sgdSSD-32
-------这一段新红旗操作系统不用加了
[root@effectiveness-db-05 ~]# vim /etc/profile
if [ $USER = "oracle" ]||[ $USER = "grid" ]; then
        if [ $SHELL = "/bin/ksh" ]; then
                ulimit -p 16384
                ulimit -n 65536
        else
                ulimit -u 16384 -n 65536
        fi
                umask 022
fi
-------
[root@effectiveness-db-05 ~]# vim /etc/security/limits.conf
grid            soft    nproc   2047
grid            hard    nproc   16384
grid            soft    nofile  1024
grid            hard    nofile  65536
oracle          soft    nproc   2047
oracle          hard    nproc   16384
oracle          soft    nofile  1024
oracle          hard    nofile  65536
[root@effectiveness-db-05 ~]# vim /etc/pam.d/login
session    required     pam_limits.so

5、创建用户和组
groupadd -g 1000 oinstall
groupadd -g 1001 dba
groupadd -g 1002 oper
groupadd -g 1003 asmadmin
groupadd -g 1004 asmdba
groupadd -g 1005 asmoper
useradd -u 2000 -g oinstall -G asmadmin,asmdba,asmoper,dba -d /home/grid grid
useradd -u 2001 -g oinstall -G asmdba,asmadmin,dba,oper -d /home/oracle oracle

passwd grid
passwd oracle

6、创建目录
mkdir -p /oracle/app/grid
mkdir -p /oracle/app/11.2.0/grid
chown -R grid:oinstall /oracle
mkdir -p /oracle/app/oracle
chown -R oracle:oinstall /oracle/app/oracle
chmod -R 775 /oracle
mkdir -p /lsi/oracle/oradata/
chown -R oracle:oinstall /lsi/oracle/oradata/
chown -R oracle:oinstall /lsi
7、设置环境变量


Oracle用户：添加如下内容
[root@effectiveness-db-05 ~]# vim /home/oracle/.bash_profile
export ORACLE_UNQNAME=tmsdb
export ORACLE_SID=tmsdb
export ORACLE_BASE=/oracle/app/oracle
export ORACLE_HOME=/oracle/app/oracle/product/11.2.0/db_1
export ORACLE_OWNER=oracle
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib:/lib:/usr/lib:/usr/local/lib
##export NLS_LANG='American_America.AL32UTF8' 
export NLS_LANG='SIMPLIFIED CHINESE_CHINA.ZHS16GBK'
export PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin
unset USERNAME
umask 022
[root@effectiveness-db-05 ~]# source /home/grid/.bash_profile
[root@effectiveness-db-05 ~]# source /home/oracle/.bash_profile




大页内存
vi /etc/sysctl.conf 

net.core.rmem_max=4194304
net.core.wmem_max=1048576
net.core.rmem_default=262144
net.core.wmem_default=262144
net.ipv4.ip_local_port_range =9000 65500
fs.file-max = 6815744
#fs.aio-max-nr =1048576
fs.aio-max-nr=4194304
kernel.shmmni =4096

#####add by Roger(hugepage)
vm.nr_hugepages = 79360


sysctl -p

vi /etc/security/limits.conf
oracle soft memlock 162529280
oracle hard memlock 162529280
=============================================================
一安装数据库软件--拷贝一份原来的响应文件--再修改对应选项
vi database/response/db_install.rsp
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=kanbanh11
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/oracle/app/oracle/
SELECTED_LANGUAGES=en,zh_CN
ORACLE_HOME=/oracle/app/oracle/product/11.2.0/db_1
ORACLE_BASE=/oracle/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.EEOptionsSelection=false
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oinstall
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
oracle.install.db.config.starterdb.globalDBName=tmsdb
oracle.install.db.config.starterdb.SID=tmsdb
oracle.install.db.config.starterdb.password.ALL=sys
oracle.install.db.config.starterdb.control=DB_CONTROL
oracle.install.db.config.starterdb.automatedBackup.enable=false
oracle.install.db.config.starterdb.storageType=FILE_SYSTEM_STORAGE
DECLINE_SECURITY_UPDATES=true
======================================================================
./runInstaller -responseFile /lsi/database/response/wty_db_install.rsp -silent -ignoreSysPrereqs

=========================================================
二、建库
dbca -silent -responseFile /lsi/database/response/gyc.rsp

[GENERAL]
RESPONSEFILE_VERSION = "11.2.0"
OPERATION_TYPE = "createDatabase"
[CREATEDATABASE]
GDBNAME = "tmsdb"
SID = "tmsdb"
TEMPLATENAME = "General_Purpose.dbc"
SYSPASSWORD = "sys"
SYSTEMPASSWORD = "sys"
DATAFILEDESTINATION = /lsi/oracle/oradata
STORAGETYPE=FS
CHARACTERSET = "ZHS16GBK"
NATIONALCHARACTERSET= "AL16UTF16"

三删除rac数据库
dbca -silent -deleteDatabase -sourcedb tmsdb 
四 创建rac数据库
dbca -silent -responseFile /home/oracle/gyc.rsp
[GENERAL]
RESPONSEFILE_VERSION = "11.2.0"
OPERATION_TYPE = "createDatabase"
[CREATEDATABASE]
GDBNAME = "tmsdb"
SID = "tmsdb"
NODELIST=sgdDB-11,sgdDB-12  -----节点hosts name 机器名字
TEMPLATENAME = "General_Purpose.dbc"
SYSPASSWORD = "sys"
SYSTEMPASSWORD = "sys"
DISKGROUPNAME=DATA1
STORAGETYPE=ASM
CHARACTERSET = "ZHS16GBK"
NATIONALCHARACTERSET= "AL16UTF16"



[GENERAL]
RESPONSEFILE_VERSION = "11.2.0"
OPERATION_TYPE = "createDatabase"
[CREATEDATABASE]
GDBNAME = "hollycrm"
SID = "hollycrm"
TEMPLATENAME = "General_Purpose.dbc"
DATAFILEDESTINATION = /oracle/oracle/oradata
STORAGETYPE=FS
CHARACTERSET = "AL32UTF8"
NATIONALCHARACTERSET= "AL16UTF16"
LISTENERS=LISTENER
TOTALMEMORY = "700"
SYSPASSWORD = "oracle"
SYSTEMPASSWORD = "oracle"
```