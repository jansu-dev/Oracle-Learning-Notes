---
title:[PG]-pgpool高可用集群部署
date:2020-11-10
---



##### p

[TOC]

##### gpool高可用集群部署

- [pgpool简介](pgpool简介)
- [软件下载与IP规划](软件下载与IP规划)
- [安装pgpool](安装pgpool)



### pgpool简介

基于PG的流复制再加上pgpool软件实现双机热备，类似于Oracle DG的FSF功能。本案例是在配置好流复制的基础上进行的，详细描述了配置过程，主备切换，性能测试，主备维护等。
拓扑结构：

pg主备节点实现流复制热备，pgpool1，pgpool2作为中间件，将主备pg节点加入集群，实现读写分离，负载均衡和HA故障自动切换。两pgpool节点可以使用一个虚拟ip节点作为应用程序访问的地址，两节点之间通过watchdog进行监控，当pgpool1宕机时，pgpool2会自动接管虚拟ip继续对外提供不间断服务。



### 软件下载与IP规划

- IP规划

| 角色        | 主机名 | IP地址         | 端口         |
| ----------- | ------ | -------------- | ------------ |
| Master      | Pg1    | 192.168.18.211 | 1922         |
| backend:0   | Pg1    | 192.168.18.211 | 9999         |
| Slave       | Pg2    | 192.168.18.212 | 1922         |
| backend:1   | Pg2    | 192.168.18.212 | 9999         |
| Vip(虚拟IP) |        | 192.168.18.215 | 对外提供服务 |

- 软件下载
  软件：pgpool-II-3.7.13
  各版本下载地址：http://pgpool.net/mediawiki/index.php/Downloads



### 安装pgpool

- 配置主机信任关系

```
--在pg1机器上都生成ssh密钥和公钥如下：
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys.pg1
chmod 600 ~/.ssh/authorized_keys.pg1

--在pg2机器上都生成ssh密钥和公钥如下：
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys.pg2
chmod 600 ~/.ssh/authorized_keys.pg2

--把pg1的公钥cp到pg2上，并且改名：
scp .ssh/authorized_keys.pg1  pg2:/home/postgres/.ssh/authorized_keys

--把pg2的公钥cp到pg1上，并且改名：
scp .ssh/authorized_keys.pg2  pg1:/home/postgres/.ssh/authorized_keys

--把pg1的公钥写入到自己主机的authorized_keys文件中，自己主机信任自己：
cat .ssh/authorized_keys.pg1 >> .ssh/authorized_keys

--把pg2的公钥写入到自己主机的authorized_keys文件中，自己主机信任自己：
cat .ssh/authorized_keys.pg2 >> .ssh/authorized_keys

--验证信任关系配置是否成功，注意远程和本机都要以远程方式验证，如果不需要密码，说明配置成功。
#pg1主机
ssh postgres@pg2 uptime
ssh postgres@pg1 uptime   #第一次需要密码

#pg2主机
ssh postgres@pg2 uptime   #第一次需要密码
ssh postgres@pg1 uptime
```

##### 3.1、安装pgpool

```
mkdir /usr/local/pgpool （root）
chown postgres:postgres /usr/local/pgpool  (root)

cd /soft/pgpool-II-3.7.13
./configure --prefix=/usr/local/pgpool --with-pgsql=/usr/local/pg12.2/
make
make install
```

##### 3.2、安装pgpool相关函数，可选，建议安装

```
cd /soft/pgpool-II-3.7.13/src/sql
make
make install
cd sql
psql -f insert_lock.sql
```

##### 3.3、配置postgres用户环境变量（pg1，pg2）

```
vi .bash_profile
export PGPOOL_HOME=/usr/local/pgpool
export PATH=$PATH:$PGPOOL_HOME/bin
```

#### 四、配置pgpool

##### 4.1、配置pg1主机上的pool_hba.conf

pool_hba.conf是对登录用户进行验证的，要和pg的pg_hba.conf保持一致。

```
cd  /usr/local/pgpool/etc/
cp  pool_hba.conf.sample  pool_hba.conf

vi  pool_hba.conf –添加如下内容
host    replication     repl            pg2                     trust
host    replication     repl            192.168.18.0/24           trust
host    all             all             192.168.18.0/24          trust
```

#####   4.2、配置pg2主机上的pool_hba.conf，添加如下内容：

```
host    replication     repl            pg1                     trust
host    replication     repl            192.168.18.0/24           trust
host    all             all             192.168.18.0/24          trust
```

##### 4.3、配置pcp.conf（pg1,pg2）

pcp.conf配置用于pgpool自己登陆管理使用的，一些操作pgpool的工具会要求提供密码等，比如节点的添加和删除等，配置如下：
cd /usr/local/pgpool/etc
cp pcp.conf.sample pcp.conf

使用pg_md5生成配置的用户名密码

```
pg_md5 postgres
e8a48653851e28c69d0506508fb27fc5
```

编辑pcp.conf文件，文件里面有样本内容

```
postgres:e8a48653851e28c69d0506508fb27fc5
```

##### 4.4、在pgpool中添加pg数据库的用户名和密码（pg1,pg2）：

需要先创建一个pgpool.conf，否则在产生pool_passwd文件时会报错：

```
cp  pgpool.conf.sample-master-slave  pgpool.conf

pg_md5 -p -m -u postgres  pool_passwd
```

\#输入数据库登录用户postgres密码，生成pool_passwd文件

##### 4.5、配置pgpool.conf

配置该文件是最核心的内容，HA是否能够正常运行，跟此文件的配置息息相关。该配置文件分为不同模块，所以我们配置的时候要根据不同模块进行分类，否则配置的时候容易出错。我们可以根据前面4.4步骤产生的配置文件进行编辑。

4.5.1、主机pg1 pgpool.conf配置，由于参数繁多，只列出需要修改或者关注的内容，有些值是默认的：

```
#------------------------------------------------------------------------------
# CONNECTIONS
#------------------------------------------------------------------------------
# - pgpool Connection Settings -
listen_addresses = '*'
port = 9999

# - pgpool Communication Manager Connection Settings -
pcp_listen_addresses = '*'
pcp_port = 9898

# - Backend Connection Settings – 此配置非常重要，不容易理解
backend_hostname0 = 'pg1'  #以后bankend0就代表了pg1在集群中的表述                              
backend_port0 = 1922       #连接pg数据库的端口                             
backend_weight0 = 1                                  
backend_data_directory0 = '/usr/local/pg12.2/data'   #pg数据库路径                                
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = 'pg2'  #bankend1代表pg2
backend_port1 = 1922
backend_weight1 = 1
backend_data_directory1 = '/usr/local/pg12.2/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

# - Authentication -
enable_pool_hba = on
pool_passwd = 'pool_passwd'  #此处的poo_passwd就是上面产生的文件


#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------
pid_file_name = '/usr/local/pgpool/pgpool.pid'


#------------------------------------------------------------------------------
# LOAD BALANCING MODE  启用负载均衡
#------------------------------------------------------------------------------
load_balance_mode = on

#------------------------------------------------------------------------------
# MASTER/SLAVE MODE
#------------------------------------------------------------------------------
master_slave_mode = on
master_slave_sub_mode = 'stream'

# - Streaming – 流复制检查
sr_check_period = 10
sr_check_user = 'repl'  #postgres replication用户的名字
sr_check_period = 0
sr_check_password = 'oracle'  #postgres replication用户的密码
sr_check_database = 'postgres'

#------------------------------------------------------------------------------
# HEALTH CHECK GLOBAL PARAMETERS  各个节点之间健康检查
#------------------------------------------------------------------------------
health_check_period = 10  #默认为0，则不检查
health_check_timeout = 20
health_check_user = 'postgres'  #pg数据库用户的名字，要求要有supper权限
health_check_password = 'postgres'
health_check_database = 'postgres'

#------------------------------------------------------------------------------
# FAILOVER AND FAILBACK
#------------------------------------------------------------------------------

failover_command = '/usr/local/pgpool/failover_stream.sh %H'  #切换脚本，后面需要编辑

#------------------------------------------------------------------------------
# WATCHDOG
#------------------------------------------------------------------------------

# - Enabling -

use_watchdog = on

# - Watchdog communication Settings -  看门狗设置

wd_hostname = 'pg1'
wd_port = 9000   #看门狗进行通信的端口

# - Virtual IP control Setting – 虚拟ip配置，非常重要，否则无法启动虚拟ip

delegate_IP = '192.168.18.215'  #虚拟ip
if_cmd_path = '/sbin'
if_up_cmd = 'ip addr add $_IP_$/24 dev eth4 label eth4:0'  #注意选择公网ip的网口
if_down_cmd = 'ip addr del $_IP_$/24 dev eth4'  #以后需要把ip命令加上setuid的权限
arping_path = '/usr/sbin' 
arping_cmd = 'arping -U $_IP_$ -w 1 -I eth4'

# - Lifecheck Setting –
wd_monitoring_interfaces_list = 'eth4'

# -- heartbeat mode – 心跳线设置，与备机的通信

wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30
heartbeat_destination0 = 'pg2'   #备机的名字
heartbeat_destination_port0 = 9694  #进行测试存活状态的端口
heartbeat_device0 = 'eth1'    #pg2网络心跳的网口，一般选择专门的网口

# - Other pgpool Connection Settings – 与pg2连接的配置

other_pgpool_hostname0 = 'pg2'  
other_pgpool_port0 = 9999
other_wd_port0 = 9000
4.5.2、备机pg2 pgpool.conf配置，可以把主机的配置复制到备机，然后做少量的修改。
#------------------------------------------------------------------------------
# CONNECTIONS
#------------------------------------------------------------------------------
# - pgpool Connection Settings -
listen_addresses = '*'
port = 9999

# - pgpool Communication Manager Connection Settings -
pcp_listen_addresses = '*'
pcp_port = 9898

# - Backend Connection Settings – 此配置非常重要，不容易理解
backend_hostname0 = 'pg1'  #以后bankend0就代表了pg1在集群中的表述                              
backend_port0 = 1922       #连接pg数据库的端口                             
backend_weight0 = 1                                  
backend_data_directory0 = '/usr/local/pg12.2/data'   #pg数据库路径                                
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = 'pg2'  #bankend1代表pg2
backend_port1 = 1922
backend_weight1 = 1
backend_data_directory1 = '/usr/local/pg12.2/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

# - Authentication -
enable_pool_hba = on
pool_passwd = 'pool_passwd'  #此处的poo_passwd就是上面产生的文件


#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------
pid_file_name = '/usr/local/pgpool/pgpool.pid'


#------------------------------------------------------------------------------
# LOAD BALANCING MODE  启用负载均衡
#------------------------------------------------------------------------------
load_balance_mode = on

#------------------------------------------------------------------------------
# MASTER/SLAVE MODE
#------------------------------------------------------------------------------
master_slave_mode = on
master_slave_sub_mode = 'stream'

# - Streaming – 流复制检查
sr_check_period = 10
sr_check_user = 'repl'  #postgres replication用户的名字
sr_check_period = 0
sr_check_password = 'oracle'  #postgres replication用户的密码
sr_check_database = 'postgres'

#------------------------------------------------------------------------------
# HEALTH CHECK GLOBAL PARAMETERS  各个节点之间健康检查
#------------------------------------------------------------------------------
health_check_period = 10  #默认为0，则不检查
health_check_timeout = 20
health_check_user = 'postgres'  #pg数据库用户的名字，要求要有supper权限
health_check_password = 'postgres'
health_check_database = 'postgres'

#------------------------------------------------------------------------------
# FAILOVER AND FAILBACK
#------------------------------------------------------------------------------

failover_command = '/usr/local/pgpool/failover_stream.sh %H'  #切换脚本，后面需要编辑

#------------------------------------------------------------------------------
# WATCHDOG
#------------------------------------------------------------------------------

# - Enabling -

use_watchdog = on

# - Watchdog communication Settings -  看门狗设置

wd_hostname = 'pg2'
wd_port = 9000   #看门狗进行通信的端口

# - Virtual IP control Setting – 虚拟ip配置，非常重要，否则无法启动虚拟ip

delegate_IP = '192.168.18.215'  #虚拟ip
if_cmd_path = '/sbin'
if_up_cmd = 'ip addr add $_IP_$/24 dev eth4 label eth4:0'  #注意选择公网ip的网口
if_down_cmd = 'ip addr del $_IP_$/24 dev eth4'  #以后需要把ip命令加上setuid的权限
arping_path = '/usr/sbin' 
arping_cmd = 'arping -U $_IP_$ -w 1 -I eth4'

# - Lifecheck Setting –
wd_monitoring_interfaces_list = 'eth4'

# -- heartbeat mode – 心跳线设置，与备机的通信

wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30
heartbeat_destination0 = 'pg1'   #主机的名字
heartbeat_destination_port0 = 9694  #进行测试存活状态的端口
heartbeat_device0 = 'eth4'    #pg1网络心跳的网口，一般选择专门的网口

# - Other pgpool Connection Settings – 与pg2连接的配置

other_pgpool_hostname0 = 'pg1'  
other_pgpool_port0 = 9999
other_wd_port0 = 9000
```

#### 五、其它辅助设置

##### 5.1、切换脚本编辑（pg1，pg2）

```
vi /usr/local/pgpool/failover_stream.sh
#! /bin/sh 
# Failover command for streaming replication. 
# Arguments: $1: new master hostname. 

new_master=$1 
trigger_command="$PG_HOME/bin/pg_ctl promote -D $PGDATA" 

# Prompte standby database. 
/usr/bin/ssh -T $new_master $trigger_command 

exit 0;
授予可执行权限：
Chmod +x /usr/local/pgpool/failover_stream.sh
```

##### 5.2、设置 ip, arping 等setuid 权限，执行failover_stream.sh需要用到，否则无法启动虚拟ip(pg1,pg2)。

```
chmod  u+s  /sbin/ip
chmod  u+s  /sbin/arping
```

#### 六 启动集群守护进程

##### 6.1、PG1启动数据库与pgpool进程：

```
pg_ctl start –l pg.log
    pgpool -n -d -D > /usr/local/pgpool/pgpool.log 2>&1 &  #-d为debug状态，显示很多内容
```

##### 6.2、PG2启动数据库与pgpool进程：

```
pg_ctl start –l pg.log
    pgpool -n -d -D > /usr/local/pgpool/pgpool.log 2>&1 &  #-d为debug状态，显示很多内容
```

##### 6.3、查看pgpool后台进程：

```
ps -ef|grep pgpool
postgres  3159  3078  0 Mar10 pts/3    00:00:00 tail -f pgpool.log
postgres  3166  3024  0 Mar10 pts/1    00:00:00 pgpool -n -D
postgres  3167  3166  0 Mar10 pts/1    00:00:02 pgpool: watchdog
postgres  3170  3166  0 Mar10 pts/1    00:00:00 pgpool: lifecheck
postgres  3187  3170  0 Mar10 pts/1    00:00:01 pgpool: heartbeat receiver
postgres  3188  3170  0 Mar10 pts/1    00:00:03 pgpool: heartbeat sender
postgres  3201  3166  0 Mar10 pts/1    00:00:00 pgpool: postgres postgres 192.168.18.212(43063) idle
postgres  3210  3166  0 Mar10 pts/1    00:00:00 pgpool: health check process(0)
postgres  3211  3166  0 Mar10 pts/1    00:00:01 pgpool: health check process(1)
postgres  3908  3166  0 Mar10 pts/1    00:00:00 pgpool: PCP: wait for connection request
postgres  3909  3166  0 Mar10 pts/1    00:00:01 pgpool: worker process
```

##### 6.4、查看后台运行日志

```
tail  -f  /usr/local/pgpool/pgpool.log  
2020-03-10 21:45:21: pid 3297: LOG:  Backend status file /tmp/pgpool_status discarded
2020-03-10 21:45:21: pid 3297: LOG:  waiting for watchdog to initialize
2020-03-10 21:45:21: pid 3298: LOG:  setting the local watchdog node name to "pg2:9999 Linux pg2"
2020-03-10 21:45:21: pid 3298: LOG:  watchdog cluster is configured with 1 remote nodes
2020-03-10 21:45:21: pid 3298: LOG:  watchdog remote node:0 on pg1:9000
2020-03-10 21:45:21: pid 3298: LOG:  watchdog node state changed from [DEAD] to [LOADING]
2020-03-10 21:45:21: pid 3298: LOG:  new outbound connection to pg1:9000 
2020-03-10 21:45:21: pid 3298: LOG:  new watchdog node connection is received from "192.168.18.211:1250"
2020-03-10 21:45:21: pid 3298: LOG:  new node joined the cluster hostname:"pg1" port:9000 pgpool_port:9999
2020-03-10 21:45:21: pid 3298: DETAIL:  Pgpool-II version:"3.7.13" watchdog messaging version: 1.1
2020-03-10 21:45:26: pid 3298: LOG:  watchdog node state changed from [LOADING] to [JOINING]
2020-03-10 21:45:26: pid 3298: LOG:  setting the remote node "pg1:9999 Linux pg1" as watchdog cluster master
```

#### 七、日常维护

##### 7.1、启动顺序

先启动主库的pg数据库，然后启动主库的pgpool守护进程，这样子vip会在主库上生成，否则会在备库产生，但是不影响业务的访问，从这可以看出vip是可以在不同的集群上漂移，跟以往的双机热备有所区别，以前vip总是在主库上产生。
接着启动备库pg数据库，然后启动pgpool进程。

##### 7.2、故障切换

7.2.1、模拟主库pgpool进程中断：

```
pgpool  –m  start  stop
```

此时备库主机的pgpool会把vip给接管过来，但是主备库的角色没有发生变化。

```
testdb=> show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | pg1      | 1922 | up     | 0.500000  | standby | 0          | true              | 0
 1       | pg2      | 1922 | up     | 0.500000  | primary | 1          | false             | 0
```

同时业务不受影响，连接正常。

```
psql  -h vip -p 9999 -U john -d testdb
```

7.2.2、模拟主库数据库关闭：

```
pg_ctl -m fast stop
此时备库的pgpool切换脚本会把备库切换成主库，提供主库服务。
检查postgresql.auto.conf文件，发现里面的primary_conninfo内容还在，需要把该行注释掉，否则不会承担主库的责任，向备库发送日志。
重启数据库。
7.2.3、把原来的主库转换成备库
创建standby.signal文件：
primary_conninfo = 'host=pg1 application_name=standby_pg2 port=1922 user=repl password=oracle options=''-c wal_sender_timeout=5000'''
restore_command = 'cp /home/postgres/arch/%f %p'
archive_cleanup_command = 'pg_archivecleanup /home/postgres/arch %r'
standby_mode = on
    修改postgresql.auto.conf文件：
primary_conninfo = 'user=repl passfile=''/home/postgres/.pgpass'' host=pg1  application_name=standby_pg2 port=1922 sslmode=disable sslcompressio
n=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
    此时日志文件报错，主备库的时间线不一致：
new timeline 8 forked off current database system timeline 7 before current recovery point 1/420000A0
    使用pg_rewind进行恢复：
pg_rewind --target-pgdata $PGDATA --source-server='host=192.168.18.211 port=1922  user=postgres dbname=testdb'

pg_rewind: servers diverged at WAL location 1/410000A0 on timeline 7
pg_rewind: rewinding from last common checkpoint at 1/41000028 on timeline 7

pg_rewind: Done!
7.2.4、启动备库
注意rewind以后会自动把standby.signal文件删除，以及修改postgres.auto.conf文件，需要手动修改。
```

实验过程中会报错，把主备库多启动几次后问题解决，可能是需要不断的调整。
后来又做了一次切换，发现没有碰到上面那么多问题，切换比较顺利。

```
总结：
Pgpool发生问题，只是到账vip发生漂移，不会影响主备库的状态，也不会影响业务。
主库数据库如果关闭或者异常中断，那么pgpool就会备库切换成主库，提供服务，不影响业务，但是恢复主备库之间的关系，需要人为的干预，不够智能。
所以以后如果要维护数据库，想不发生主备库切换，需要先停止pgpool进程，然后再关闭主库，否则会发生切换，还得修复备库，增加工作量。
```



### 压力测试

##### 8.1、通过第三台机器网络连接测试：

```
more yalitest.sh 
#!/bin/bash

BEGIN_TIME=`date`

for i in {1..10000} 
do 
#echo $i  
psql -p 9999 -U postgres -h vip -d postgres -c "SELECT * FROM pgbench_accounts WHERE aid = $i"  
done

END_TIME=`date`

echo 'BEGIN TIME IS ' $BEGIN_TIME 'END_TIME IS ' $END_TIME

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | pg1      | 1922 | up     | 0.500000  | standby | 698745     | false             | 0
 1       | pg2      | 1922 | up     | 0.500000  | primary | 2005824    | true              | 0
```

##### 8.2、pgbench工具测试

--初始化数据：

```
pgbench -i testdb

pgbench -c 30 -T 60 -h vip -p 9999  -U postgres -r postgres

starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 30
number of threads: 1
duration: 60 s
number of transactions actually processed: 22493
latency average = 80.087 ms
tps = 374.591718 (including connections establishing)
tps = 374.608719 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
         0.001  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
         2.166  BEGIN;
         8.840  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         2.260  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         3.446  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         9.520  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         2.383  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
        51.283  END;
该测试用例只会在主库上操作，因为是dml操作，所以只能连接到主库。
postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | pg1      | 1922 | up     | 0.500000  | standby | 414026     | false             | 99424
 1       | pg2      | 1922 | up     | 0.500000  | primary | 572956     | true              | 0
```

注意replication_delay有了值，经过观察是备库做同步的时候延迟行数，只有在测试过程中才会出现，等同步完成就会归零。

##### 8.3、进行select压力测试，2个节点负载均衡：

```
pgbench -c 30 -T 60 -S -h vip -p 9999  -U postgres -r postgres

starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 30
number of threads: 1
duration: 60 s
number of transactions actually processed: 188317
latency average = 9.565 ms
tps = 3136.558834 (including connections establishing)
tps = 3136.724522 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
         9.524  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;

duration: 60 s
number of transactions actually processed: 169220
latency average = 10.653 ms
tps = 2816.036003 (including connections establishing)
tps = 2816.167417 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        10.616  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;

duration: 60 s
number of transactions actually processed: 192090
latency average = 9.374 ms
tps = 3200.446677 (including connections establishing)
tps = 3200.575641 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
         9.337  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

##### 8.4、把备库脱离出集群，进行单机测试:

```
pcp_detach_node -h vip  -p 9898 -U postgres -n 0
    pgbench -c 30 -T 60 -S -h vip -p 9999  -U postgres -r postgres
    通过观察select_cnt，备库统计没有发生变化。
duration: 60 s
number of transactions actually processed: 148438
latency average = 12.136 ms
tps = 2471.925771 (including connections establishing)
tps = 2472.039335 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        12.097  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 192639
latency average = 9.351 ms
tps = 3208.339596 (including connections establishing)
tps = 3208.466385 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
         9.317  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 197665
latency average = 9.113 ms
tps = 3291.849823 (including connections establishing)
tps = 3291.959408 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
         9.084  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
对比没有发现负载均衡起到很大的作用。
```

##### 8.5、增加并发用户，提高压力测试

```
pgbench -c 60 -T 60 -S -h vip -p 9999  -U postgres -r postgres

duration: 80 s
number of transactions actually processed: 266983
latency average = 18.006 ms
tps = 3332.161044 (including connections establishing)
tps = 3332.301769 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        17.905  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 80 s
number of transactions actually processed: 271234
latency average = 17.714 ms
tps = 3387.202774 (including connections establishing)
tps = 3387.367938 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        17.604  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
分离一个节点继续测试：
pcp_detach_node -h vip  -p 9898 -U postgres -n 0
    pgbench -c 60 -T 80 -S -h vip -p 9999  -U postgres -r postgres
duration: 80 s
number of transactions actually processed: 240180
latency average = 20.015 ms
tps = 2997.799781 (including connections establishing)
tps = 2997.904556 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        19.850  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 80 s
number of transactions actually processed: 251094
latency average = 19.149 ms
tps = 3133.313696 (including connections establishing)
tps = 3133.490651 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        18.999  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 80 s
number of transactions actually processed: 263386
latency average = 18.252 ms
tps = 3287.288776 (including connections establishing)
tps = 3287.467205 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        18.135  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
    加入节点：
pcp_attach_node -h vip -p 9898 -U postgres -n 0
```



##### 8.6、连接单机主库进行测试：

```
pgbench -c 30 -T 60 -S -h pg2 -p 1922  -U postgres -r postgres
```

第一遍：

```
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 30
number of threads: 1
duration: 60 s
number of transactions actually processed: 78748
latency average = 22.880 ms
tps = 1311.215262 (including connections establishing)
tps = 1311.247768 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        22.848  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

第二遍：

```
duration: 60 s
number of transactions actually processed: 80374
latency average = 22.413 ms
tps = 1338.480191 (including connections establishing)
tps = 1338.512735 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        22.385  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

第三遍：

```
duration: 60 s
number of transactions actually processed: 78871
latency average = 22.840 ms
tps = 1313.458523 (including connections establishing)
tps = 1313.492331 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        22.811  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

##### 8.7、连接vip进行负载均衡测试：

```
postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | pg1      | 1922 | up     | 0.500000  | standby | 711909     | false             | 8440
 1       | pg2      | 1922 | up     | 0.500000  | primary | 2015619    | true              | 0
```

Select_cnt统计不断发生变化，说明两个节点都被分配了查询任务，实现负载均衡。

第一遍：

```
pgbench -c 30 -T 60 -S -h vip -p 9999  -U postgres -r postgres

starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 30
number of threads: 1
duration: 60 s
number of transactions actually processed: 136558
latency average = 13.195 ms
tps = 2273.668774 (including connections establishing)
tps = 2273.778799 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        13.155  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

第二遍：

```
duration: 60 s
number of transactions actually processed: 130250
latency average = 13.829 ms
tps = 2169.355643 (including connections establishing)
tps = 2169.458627 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        13.791  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

第三遍：

```
duration: 60 s
number of transactions actually processed: 156074
latency average = 11.540 ms
tps = 2599.621368 (including connections establishing)
tps = 2599.749783 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        11.508  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

第四遍：

```
duration: 60 s
number of transactions actually processed: 155708
latency average = 11.571 ms
tps = 2592.586873 (including connections establishing)
tps = 2592.706989 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        11.539  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

通过分析，发现通过网络连接到单机进行查询时所花平均时间为22秒；但是通过网络连接到vip（集群）进行查询时，所花平均时间为12秒，可以看出负载均衡还是起到了作用。单机的tps平均为1313，集群的tps为2592。

如果在本机进行压力测试，结果是比集群的要快很多，可能在网络上会有很多的消耗。

```
pgbench -c 30 -T 60 -S -h pg2 -p 1922  -U postgres
latency average = 2.614 ms
tps = 11474.575733 (including connections establishing)
tps = 11474.786866 (excluding connections establishing)

    8.8、pgpool.conf中与连接数量相关的参数：
#------------------------------------------------------------------------------
# POOLS
#------------------------------------------------------------------------------

# - Concurrent session and pool size -

num_init_children = 120  #后台产生等待连接的进程
                      #  ps -ef|grep request
                      # pgpool: wait for connection request
max_pool = 4
```

\#在 pgpool-II 子进程中缓存的最大连接数。当有新的连接使用相同的用户名连接到相同的数据库，pgpool-II 将重用缓存的连接。如果不是，则 pgpool-II 建立一个新的连接到 PostgreSQL。如果缓存的连接数达到了 max_pool，则最老的连接将被抛弃，并使用这个槽位来保存新的连接。默认值为 4。
如果该参数设置太少的话，用下面的测试用例只能支持最多30个并发连接：

```
8.9、调整pg数据库的最大连接数为1000，然后再测试：
两个节点负载均衡：
pgbench -c 100 -T 60 -S -h vip -p 9999  -U postgres -r postgres
duration: 60 s
number of transactions actually processed: 97337
latency average = 61.851 ms
tps = 1616.801225 (including connections establishing)
tps = 1616.907355 (excluding connections establishing)
statement latencies in milliseconds:
         0.003  \set aid random(1, 100000 * :scale)
        61.087  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 90861
latency average = 61.344 ms
tps = 1630.148983 (including connections establishing)
tps = 1630.231327 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        60.663  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 105577
latency average = 57.013 ms
tps = 1753.979544 (including connections establishing)
tps = 1754.060665 (excluding connections establishing)
statement latencies in milliseconds:
         0.003  \set aid random(1, 100000 * :scale)
        56.487  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 108602
latency average = 55.300 ms
tps = 1808.315176 (including connections establishing)
tps = 1808.403380 (excluding connections establishing)
statement latencies in milliseconds:
         0.003  \set aid random(1, 100000 * :scale)
        54.867  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
一个节点：
duration: 60 s
number of transactions actually processed: 101261
latency average = 59.440 ms
tps = 1682.356283 (including connections establishing)
tps = 1682.430350 (excluding connections establishing)
statement latencies in milliseconds:
         0.003  \set aid random(1, 100000 * :scale)
        58.966  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 100418
latency average = 59.951 ms
tps = 1668.027917 (including connections establishing)
tps = 1668.093257 (excluding connections establishing)
statement latencies in milliseconds:
         0.003  \set aid random(1, 100000 * :scale)
        59.473  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 132744
latency average = 45.252 ms
tps = 2209.859230 (including connections establishing)
tps = 2209.937983 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        45.009  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;

duration: 60 s
number of transactions actually processed: 76405
latency average = 78.842 ms
tps = 1268.357245 (including connections establishing)
tps = 1268.400040 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
        78.098  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
duration: 60 s
number of transactions actually processed: 112301
latency average = 53.503 ms
tps = 1869.039670 (including connections establishing)
tps = 1869.105436 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        53.226  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```