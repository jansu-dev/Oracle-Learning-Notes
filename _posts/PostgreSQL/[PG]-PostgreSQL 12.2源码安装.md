---
title:[PG]-PostgreSQL 12.2源码安装
date:2020-11-11
---



##### PostgreSQL 12源码安装

- [用户与环境配置](用户与环境配置)

- [环境变量配置](环境变量配置)

- [内核参数配置](内核参数配置)

- [源代码安装](源代码安装)

- [配置可选项](配置可选项)

- [数据库实例](数据库实例)

- [编辑配置文件](编辑配置文件)

  

### 用户与环境配置

创建用户

```
用户与环境配置
# groupadd postgres
# useradd -g postgres postgres
```



### 环境变量配置

```
export PGPORT=1922
export PG_HOME=/usr/local/pg12.2
export PATH=$PG_HOME/bin:$PATH
export PGDATA=$PG_HOME/data
export LD_LIBRARY_PATH=$PG_HOME/lib
export LANG=en_US.utf8
```



### 内核参数配置

```
vi /etc/sysctl.conf
内核参数配置
kernel.shmmax = 68719476736（默认） #最大共享内存段大小
kernel.shmall = 4294967296（默认） #可以使用的共享内存的总量
kernel.shmmni = 4096 #整个系统共享内存段的最大数目
kernel.sem = 50100 64128000 50100 1280 #每个信号对象集的最大信号对象数
fs.file-max = 7672460 #文件句柄的最大数量。
net.ipv4.ip_local_port_range = 9000 65000 #应用程序可使用的IPv4端口范围
net.core.rmem_default = 1048576 #套接字接收缓冲区大小的缺省值
net.core.wmem_default = 262144 #套接字发送缓冲区大小的缺省值
net.core.wmem_max = 1048576 #套接字发送缓冲区大小的最大值


# sysctl -p
```



### 源代码安装

--使用postgres用户安装

```
cd /soft/postgresql-12.2
./configure --prefix=/usr/local/pg12.2
make
make install
```



### 配置可选项

```
./configure --prefix=/usr/local/pg12.2 --with-pgport=1922 --with-openssl --
with-perl --with-tcl --with-python --with-pam --without-ldap --with-libxml --
with-libxslt --enable-thread-safety --with-wal-blocksize=16 --with-blocksize=8 --
enable-dtrace --enable-debug
```



### 数据库实例

```
--创建目录
mkdir /usr/local/pg12.2/data
--初始化数据库集簇
initdb -D $PGDATA –w --data-checksums –复制时需要
--启动数据库集簇
pg_ctl -D $PGDATA -l logfile start
--创建新的数据库
createdb test
--登录数据库
psql test
```



### 编辑配置文件

编辑pg_hba.conf 配置文件

```
vi $PGDATA/pg_hba.conf

# TYPE DATABASE USER ADDRESS METHOD
# "local" is for Unix domain socket connections only
local all all trust
# IPv4 local connections:
host all all 127.0.0.1/32 trust
host all all 192.168.18.0/24 md5
host all all pg2 reject
```

编辑postgresql.conf 配置文件

```
vi $PGDATA/postgresql.conf

listen_addresses = ‘*' # what IP address(es) to listen on;
# comma-separated list of addresses;
# defaults to 'localhost'; use '*' for all
# (change requires restart)
port = 1922 # (change requires restart)
max_connections = 100
```