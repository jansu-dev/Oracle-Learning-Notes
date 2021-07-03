---
title:[PG]-PostgreSQL流复制配置
date:2020-11-10
---





##### [PG]-PostgreSQL流复制配置

- [环境与IP规划](环境与IP规划)

- [创建流复制](创建流复制)

- [验证流复制](验证流复制)

- [主从切换](主从切换)

- [pg_rewind工具](pg_rewind工具)

- [实时同步](实时同步)

- [添加节点](添加节点)

- [其它配置](其它配置)

- [常见问题解决](常见问题解决)

  

### 环境与IP规划

- 环境说明

PG版本：用源码编译安装的12.2版本

- 主备机器规划主机名

| 角色   | 主机名 | ip             |
| ------ | ------ | -------------- |
| Maswer | Pg1    | 192.168.18.211 |
| Slave  | Pg2    | 192.168.18.212 |



### 创建流复制

- 设置host
  master,slave两节点都要操作。

```
# vim /etc/hosts
#编辑内容如下：
192.168.18.211  pg1
192.168.18.212  pg2
```

- 关闭防火墙

  ```
  service iptables status  
  service iptables stop  
  chkconfig iptables off  
  ```

- 在主库初始化数据库

初始化DB

```
$ initdb -D  /usr/local/pg12.1/data –U postgres
```

启动数据库

```
$pg_ctl -D /usr/local/ pg12.1/data  start
```

创建用户
格式说明：create role 同步用的用户名 login replication encrypted password '密码';

```
postgres=# create role repl login replication encrypted password 'oracle';
CREATE ROLE    

postgres=#\q
```

配置$PGDATA/data/pg_hba.conf，添加下面内容

格式：host replication 同步用的用户名 备库IP地址或域名/24 trust

```
host    replication     repl            pg2                     trust
host    replication     repl            192.168.18.0/24         trust
host    all             all             192.168.18.0/24         trust
```

配置主备库的postgres.conf文件，因为Failover时要进行角色切换，所有需要参数一样。
主库配置 ~/data/postgres.conf修改成以下内容

```
listen_addresses = '*'
wal_level = replica  --10以后的版本为replica 主从设置为在线模式，流复制必选
max_wal_senders=10  --流复制允许连接进程，主备库这个参数值必须一样
wal_keep_segments =64
archive_mode = on  -- 设置归档模式
archive_command = 'cp %p /home/postgres/arch/%f'  --设置归档cp命令

listen_addresses = '*'
wal_level = replica
max_wal_senders=20
wal_keep_segments =64

archive_mode = on
archive_command = 'cp %p /home/postgres/arch/%f'
restore_command = 'cp /home/postgres/arch/%f %p'
recovery_target_timeline = 'latest'

full_page_writes = on
wal_log_hints = on
```

重启主库服务更新配置

```
$pg_ctl -D ~/data/ -l ~/log/pglog.log restart
```

- 在备库初始化数据库

不需要初始化，直接从主库备份就行，如有DATA直接删掉或改名掉:

```
$ pg_basebackup -h pg1 -p 1922 -U repl -R -F p -P -D $PG_HOME/data


备注：
        -h，主库主机，-p，主库服务端口；
        -U，复制用户；
        -F，p是默认输出格式，输出数据目录和表空间相同的布局，t表示tar格式输出；
        -P，同--progress，显示进度；
        -D，输出到指定目录；
    -R 创建一个recovery.conf文件，10版本后就没有该文件，改为standby.signal文件，需要自己创建，所以该参数可以省略
```

备库修改配置文件，添加以下内容
postgres@NanoPI-006:~$vi ~/data/postgresql.conf

```
listen_addresses = '*'
wal_level = replica
max_wal_senders=20
wal_keep_segments =64

archive_mode = on
archive_command = 'cp %p /home/postgres/arch/%f'
restore_command = 'cp /home/postgres/arch/%f %p'
recovery_target_timeline = 'latest'

full_page_writes = on
wal_log_hints = on

hot_standby = on                           #在备份的同时允许查询，默认值        max_standby_streaming_delay = 30s            #可选，流复制最大延迟        
wal_receiver_status_interval = 10s            #可选，从向主报告状态的最大间隔时间    
hot_standby_feedback = on                    #可选，查询冲突时向主反馈        
```

配置~/data/pg_hba.conf添加下面内容；
在备库中维护的主库IP地址是为了以后切换使用。

```
host    replication     repl            pg1                     trust
host    replication     repl            192.168.18.0/24         trust
host    all             all             192.168.18.0/24         trust
```

创建备库文件standby.signal

```
primary_conninfo = 'host=pg1 port=1922 user=repl password=oracle options=''-c wal_sender_timeout=5000'''
restore_command = 'cp /home/postgres/arch/%f %p'
archive_cleanup_command = 'pg_archivecleanup /home/postgres/arch %r'
standby_mode = on

第一行参数：#连接到主库信息
第二行参数：将来变成主库时需要用到的参数。
第三行参数：变成主库后需要清空的归档日志。
第四行参数：把备库变成read-only transaction模式，不需要进行写操作。允许查询。这一点非常好。
```

启动备库数据服务

```
$pg_ctl -D $PGDATA  -l  ~/log/pglog.log  start
```



### 验证流复制

##### 3.1、

观察主从两库的归档日志的位置，或者主库两边的pg_wal目录下的内容，发现主库日志切换后，备库pg_wal目录下就会产生新的日志文件，但是在备库的归档目录下没有内容，应该是主库的归档日志传递到备库的pg_wal目录下了。

##### 3.2、

主库修改后，日志没有归档，但是备库已经同步了，类似于oracle同步时用lgwr方式进行写standby_logfile进行同步。

##### 3.3、查看当前备库状态：

```
testdb=# select pg_is_in_recovery();  
 pg_is_in_recovery 
-------------------
 t

t ：true，意味着处于recovery状态
f ：false，意味着处于正常服务状态
主库查询：
testdb=# \x
testdb=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 3711
usesysid         | 16384
usename          | repl
application_name | walreceiver
client_addr      | 192.168.18.212
client_hostname  | pg2
client_port      | 49206
backend_start    | 2020-03-03 22:08:47.924435-05
backend_xmin     | 
state            | streaming
sent_lsn         | 0/210000D8
write_lsn        | 0/210000D8
flush_lsn        | 0/210000D8
replay_lsn       | 0/210000D8
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2020-03-03 22:13:02.990258-05
#application_name 很重要，以后同步复制需要用到。
```

##### 3.4、备库数据库日志内容：

```
cp: cannot stat `/home/postgres/arch/00000002.history': No such file or directory
cp: cannot stat `/home/postgres/arch/000000010000000000000009': No such file or directory
2020-02-29 04:48:45.734 EST [4938] FATAL:  could not connect to the primary server: could not connect to server: Connection refused
                Is the server running on host "pg1" (192.168.18.211) and accepting
                TCP/IP connections on port 1922?
cp: cannot stat `/home/postgres/arch/00000002.history': No such file or directory
cp: cannot stat `/home/postgres/arch/000000010000000000000009': No such file or directory
2020-02-29 04:48:50.747 EST [4941] LOG:  started streaming WAL from primary at 0/9000000 on timeline 1
    如果主库关闭，备库数据库日志内容：
cp: cannot stat `/home/postgres/arch/00000002.history': No such file or directory
cp: cannot stat `/home/postgres/arch/00000001000000000000000C': No such file or directory
2020-02-29 05:22:55.757 EST [5048] FATAL:  could not connect to the primary server: could not connect to server: Connection refused
                Is the server running on host "pg1" (192.168.18.211) and accepting
                TCP/IP connections on port 1922?
cp: cannot stat `/home/postgres/arch/00000002.history': No such file or directory
```

- 主库后台进程：

```
ps -ef|grep "wal"
postgres  3753  3749  0 21:21 ?        00:00:00 postgres: walwriter           
postgres  3844  3749  0 21:49 ?        00:00:00 postgres: walsender repl 192.168.18.212(33595) streaming 0/8000148
```

- 备库后台进程
  一个进程负责接收，一个负责recovery

```
ps -ef|grep postgres
postgres  3472  3471  0 21:49 ?        00:00:00 postgres: startup   recovering 000000010000000000000008
postgres  3475  3471  0 21:49 ?        00:00:00 postgres: checkpointer        
postgres  3476  3471  0 21:49 ?        00:00:00 postgres: background writer   
postgres  3478  3471  0 21:49 ?        00:00:00 postgres: stats collector     
postgres  3479  3471  0 21:49 ?        00:00:00 postgres: walreceiver   streaming 0/8000148
```



### 主从切换

停掉主库

```
pg_ctl  stop  -m fast
```

执行以下命令进行主从切换，把备库改成主库，执行之后发现standby.signal被删除了：

```
pg_ctl promote
```

查看最新状态：

```
pg_controldata | grep cluster 
Database cluster state:               in production
```

##### 4.3、

这一步非常关键，注意原来的备库的postgresql.auto.conf文件中会自动添加一行primary_conninfo的信息，要把这一行给注释掉，否则虽然现在是主库了，但是配置还是当作备库，自相矛盾，且在跟踪日志中会报“background worker "logical replication launcher" (PID 6304) exited with exit code 1”错误。这可能是PG12.2的bug。

postgresql.auto.conf文件内容如下，注意下面内容只是一行数据，/home/postgres/.pgpass其实没有没有这个文件，不需要创建：

```
primary_conninfo = 'user=repl passfile=''/home/postgres/.pgpass'' host=pg1 port=1922 sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
```

重启数据库，查看后台进程，实验发现walsender进程要等备库正常启动后才会启动，备库关闭时该进程也自动中断：

```
ps -ef|grep postgres |grep -v sshd |grep -v bash
postgres  3215  3164  0 Feb29 pts/3    00:00:00 tail -f pg_log
postgres  6329     1  0 07:08 ?        00:00:00 /usr/local/pg12.2/bin/postgres
postgres  6331  6329  0 07:08 ?        00:00:00 postgres: checkpointer        
postgres  6332  6329  0 07:08 ?        00:00:00 postgres: background writer   
postgres  6333  6329  0 07:08 ?        00:00:00 postgres: walwriter           
postgres  6334  6329  0 07:08 ?        00:00:00 postgres: autovacuum launcher   
postgres  6335  6329  0 07:08 ?        00:00:00 postgres: archiver            
postgres  6336  6329  0 07:08 ?        00:00:00 postgres: stats collector     
postgres  6337  6329  0 07:08 ?        00:00:00 postgres: logical replication launcher   
postgres  6353  6329  0 07:12 ?        00:00:00 postgres: walsender repl 192.168.18.212(33609) streaming 0/1A01C7F8
```

##### 4.4、在新备库上创建一个standby.signal文件，添加如下内容：

```
primary_conninfo = 'host=pg1 port=1922 user=repl password=oracle options=''-c wal_sender_timeout=5000'''
restore_command = 'cp /home/postgres/arch/%f %p'
archive_cleanup_command = 'pg_archivecleanup /home/postgres/arch %r'
standby_mode = on
```

##### 4.5、在新备库的postgresql.auto.conf文件中添加如下内容，这一步非常关键，第一次搭建备库的时候会自动添加，但是切换后却不能：

```
primary_conninfo = 'user=repl passfile=''/home/postgres/.pgpass'' host=pg2 port=1922 sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
```

<span style=color:red>注意：/home/postgres/.pgpass其实没有没有这个文件，不需要创建。</span>

##### 4.6、启动新备库：

```
pg_ctl start -l pg_log
```

##### 4.7、查看后台进程：

```
ps -ef|grep postgre |grep -v ssh |grep -v bash
postgres  3274  3237  0 Feb29 pts/3    00:00:00 tail -f pg_log
postgres  6441     1  0 07:12 ?        00:00:00 /usr/local/pg12.2/bin/postgres
postgres  6442  6441  0 07:12 ?        00:00:00 postgres: startup   recovering 00000003000000000000001A
postgres  6447  6441  0 07:12 ?        00:00:00 postgres: checkpointer        
postgres  6448  6441  0 07:12 ?        00:00:00 postgres: background writer   
postgres  6450  6441  0 07:12 ?        00:00:00 postgres: stats collector     
postgres  6451  6441  0 07:12 ?        00:00:05 postgres: walreceiver   streaming 0/1A01C7F8
```

##### 4.8、验证主备库是否能够同步

在主库进行dml操作，发现备库能够正常同步，切换成功。
总结：
经过实验，发现主备切换不太灵活和智能，需要后续进行手动修改，特别是postgresql.auto.conf文件中自动添加的一行，在主备切换的时候不会自动删除，没有相关文档，造成了隐性的问题，给DBA造成了很大的麻烦，不容易故障排除。
主库在正常运行中，备库可以随意切换为主库，没有一个制约机制，感觉不严谨，此时变成两个主库，数据无法同步。如果此时两边的数据库都各自发生变化，将来想把一台主库当作备库，则需要在备库上对当前的数据进行同步，然后就可以变成备库，用以下的命令进行同步：

```
pg_rewind --target-pgdata $PGDATA --source-server='host=192.168.18.211 port=1922  user=postgres dbname=testdb'
pg_rewind: servers diverged at WAL location 0/1C01F280 on timeline 5
pg_rewind: rewinding from last common checkpoint at 0/1C01F1D0 on timeline 5
pg_rewind: Done!
```



### pg_rewind工具

如果备库是意外崩溃，如果新的主库修改了数据，经过的时间很长，归档日志又删除了，无法同步，原来的数据库如果想变成备库，需要对数据库做一次同步，那么就可以用到pg_rewind工具进行同步。
pg_rewind—使一个PostgreSQL数据目录与另一个数据目录（该目录从第一个PostgreSQL数据目录创建而来）一致。
描述
pg_rewind是一个在集群的时间线参数偏离之后，用于使一个PostgreSQL集群与另一个相同集群的拷贝同步的工具。一个典型的场景是在故障转移之后，让一个老的主服务器重新在线作为一个standby跟随新主服务器。
其结果相当于使用源数据目录替换目标数据目录。所有的文件都被拷贝，包括配置文件。与做一个基础备份或者像rsync这样的工具相比，pg_rewind的优势是pg_rewind不需要读取所有集群中没有更改的文件。当数据库很大，并且只有一小部分不同的集群之间，使它的速度快得多。
pg_rewind检查源集群与目标集群的时间线历史来检测它们产生分歧的点，并希望在目标集群的pg_xlog目录找到WAL回到分歧点的所有方式。在典型的故障转移场景：目标集群在分歧之后立即被关闭，那是没有问题的，但是，如果目标集群在分歧之后运行了很长一段时间，老的WAL文件可能不存在了。在这种情况下，它们可以手动从WAL归档复制到pg_xlog目录。目前不支持从一个WAL归档中自动获取丢失的文件。
在运行pg_rewind之后，当目标服务器第一次被启动，它将进入恢复模式并重放从分歧点之后源服务器产生的所有WAL。当pg_rewind被运行时，如果一些 WAL在源服务器上不再可用，因此不能用pg_rewind回话复制，当目标服务器被启动时时可以的。这可以通过在目标数据目录创建一个带有合适的restore_command命令的recovery.conf文件来实现。

选项
pg_rewind 接受下列命令行参数：
-D 目录
--target-pgdata=目录
该选项指定与源同步的目标数据目录。
--source-pgdata=目录
指定源服务器的数据目录的路径，以使目标数据目录与之同步。当—source-pgdata被使用时，源服务器必须被关闭。
--source-server=连接字符串
指定一个libpq连接字符串以连接到源PostgreSQL服务器来使目标同步。服务器必须开启并允许，并且不能处于恢复模式。
-n
--dry-run
做除了修改目标目录的所有事情。
-P
--progress
开启进程报告。在从源集群复制数据时，打开这个功能将提供一个近似的进度报告。
--debug
打印详细的调试输出对开发者调试pg_rewind来说是非常有用的。
-V
--version
显示版本信息并退出。
-?
--help
显示帮助，然后退出
环境
当—source-server选项被使用时，pg_rewind也使用libpq支持的环境变量（见31.14节）。
<span style=color:red>注意</span>
pg_rewind需要启用postgresql.conf中的wal_log_hints 选项，或者当集群被使用initdb初始化时启用数据校验。full_page_writes也必须启用。
pg_rewind是如何工作的
基本的思想是从新的集群拷贝所有的东西到老的集群，除了我们知道的相同的（数据）块。
1.从最后一个检查点开始扫描老集群的WAL日志，在该检查点之前，新集群的时间线历史从老集群被创建出来。对于每一个WAL记录，做一个数据块被触及的记录。在新的集群被创建出来以后，这产生所有在老集群中被更改的数据块的列表。
2.从新集群复制所有这些被更改的数据块到老集群。
3.从新集群复制所有其它像clog，conf这样的文件等等到老集群。每个文件，除了表文件。
4.从新集群应用WAL，从故障转移创建的检查点开始。（严格的说，pg_rewind不应用WAL，它只是创建一个备份标签文件以表明PostgreSQL被启动了，它会从检查点重放并应用所有需要的WAL）

```
2020-02-28 01:58:35.974 EST [16990] LOG:  received promote request
2020-02-28 01:58:35.974 EST [16990] LOG:  redo done at 0/50000028
2020-02-28 01:58:35.977 EST [16990] LOG:  last completed transaction was at log time 2020-02-27 21:40:31.673922-05
cp: cannot stat `/home/postgres/arch/000000090000000000000050': No such file or directory
cp: cannot stat `/home/postgres/arch/0000000A.history': No such file or directory
2020-02-28 01:58:35.987 EST [16990] LOG:  selected new timeline ID: 10
2020-02-28 01:58:36.090 EST [16990] LOG:  archive recovery complete
cp: cannot stat `/home/postgres/arch/00000009.history': No such file or directory
2020-02-28 01:58:36.112 EST [16989] LOG:  database system is ready to accept connections
```



### 实时同步

上面的配置是异步同步，对于主库的性能影响是最小的，但是会丢数据，我们可以把复制配置成实时同步。
当设置同步复制时，尽量记住以下几点：
• 最小化延迟
• 确保您有冗余延迟
• 同步复制比异步复制代价更高
同步时是通过一个关键的参数application_name来实现的。
5.1、配置主库postgres.conf，添加如下内容：
synchronous_standby_names = 'standby_pg2'
synchronous_commit = on --默认值，可以设置为remote_write，对主库性能有利

##### 5.2、重启主库

##### 5.3、修改备库standby.signal配置文件，在原来的内容中添加application_name内容：

```
primary_conninfo = 'host=pg1 application_name=standby_pg2 port=1922 user=repl password=oracle options=''-c wal_sender_timeout=5000'''
restore_command = 'cp /home/postgres/arch/%f %p'
archive_cleanup_command = 'pg_archivecleanup /home/postgres/arch %r'
standby_mode = on
```

##### 5.4、修改备库postgresql.auto.conf，添加application_name内容，实际上备库是以这个文件为主，上面修改的standby.signal并不生效：

```
primary_conninfo = 'user=repl passfile=''/home/postgres/.pgpass'' host=pg1  application_name=standby_pg2 port=1922 sslmode=disable sslcompression
=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any'
    5.5、重启备库，查看后台日志信息：
consistent recovery state reached at 0/21000188
invalid record length at 0/210001C0: wanted 24, got 0
database system is ready to accept read only connections
started streaming WAL from primary at 0/21000000 on timeline 6
    5.6、在主库查看同步状态：
postgres=# \x  --以行形式显示，类似于mysql的\G
postgres=# SELECT * FROM pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 3732
usesysid         | 16384
usename          | repl
application_name | standby_pg2
client_addr      | 192.168.18.212
client_hostname  | pg2
client_port      | 49207
backend_start    | 2020-03-03 22:14:24.010759-05
backend_xmin     | 
state            | streaming
sent_lsn         | 0/210001C0
write_lsn        | 0/210001C0
flush_lsn        | 0/210001C0
replay_lsn       | 0/210001C0
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 1
sync_state       | sync
reply_time       | 2020-03-03 22:14:44.126791-05
状态显示为实时同步。
```

##### 5.7、验证：

```
在同步过程中，如果把备库给关闭，然后在主库进行数据操作，会发现无法操作，该事务会挂起，处于等待状态。此时对主库会造成很大的影响，跟oracle的最大保护模式一样。
```

##### 5.8、如果我们配置了多个备库，而且进行实时同步，假如只要保证前面的备库能够实时就可以，那么可以进行如下设置：

```
synchronous_standby_names = 'FIRST 2 (s1, s2, s3)'
```

5.9、如果只要保证其中任何的备库同步成功，可以进行如下设置：

```
synchronous_standby_names = 'ANY 2 (s1, s2, s3)'
```



### 添加节点

##### 6.1、添加新的节点跟第二个节点添加方式一样，修改standby.signal和postgres.auto.conf文件，然后启动节点三。

##### 6.2、修改主库的postgres.conf，添加如下一行：

synchronous_standby_names = 'FIRST 2 (standby_pg2,standby_pg3)'

##### 6.3、重启主库，查看复制状态：

```
testdb=# \x
Expanded display is on.
testdb=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 8604
usesysid         | 16384
usename          | repl
application_name | standby_pg3
client_addr      | 192.168.18.213
client_hostname  | pg3
client_port      | 34436
backend_start    | 2020-03-06 05:46:23.01908-05
backend_xmin     | 
state            | streaming
sent_lsn         | 0/2A00FA38
write_lsn        | 0/2A00FA38
flush_lsn        | 0/2A00FA38
replay_lsn       | 0/2A00FA38
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 2
sync_state       | sync
reply_time       | 2020-03-06 05:46:33.088474-05
-[ RECORD 2 ]----+------------------------------
pid              | 4716
usesysid         | 16384
usename          | repl
application_name | standby_pg2
client_addr      | 192.168.18.212
client_hostname  | pg2
client_port      | 50026
backend_start    | 2020-03-06 03:13:46.966522-05
backend_xmin     | 
state            | streaming
sent_lsn         | 0/2A00FA38
write_lsn        | 0/2A00FA38
flush_lsn        | 0/2A00FA38
replay_lsn       | 0/2A00FA38
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 1
sync_state       | sync
reply_time       | 2020-03-06 05:46:27.970934-05
```

##### 6.4、验证同步

主要备库的任何一个节点无法同步，都会影响主库的事务操作。但是发现正常的一个备库节点能够同步，即使主库处于停留状态，由此证明主库已经把事务传递到备库了，只是有备库没有同步，所以处于等待状态。

##### 6.5、如果把主库的参数修改如下：

```
synchronous_standby_names = 'FIRST 1 (standby_pg2,standby_pg3)'
```

##### 6.6、实验证明，如果第二个备库节点发生故障无法同步，不会影响主库事务操作。



### 其它配置

```
7.1、正常情况下备库会尽快恢复来自于主服务器的 WAL 记录。但是有时候备库的复制延迟一段时间，它能提供机会纠正数据丢失错误。虽然这种需求比较少见，但是也有个别的需求，recovery_min_apply_delay参数允许你将复制延迟一段时间，默认时间单位则为毫秒。例如，如果你设置这个参数为10min，对于一个事务提交，只有备库的系统时间超过主库的提交时间至少 5分钟时，备库才会应用该事务。
在备库的postgresql.auto.conf添加如下参数，备库延迟recovery：
recovery_min_apply_delay = 1min

重启数据库生效，就会发现备库延迟一分钟recovery，注意这个参数只是控制备库延迟应用日志，不影响主库传输日志到备库，即使主备库配置成实时同步，不会影响主库事务操作。
7.2、如果设置了synchronous_commit=remote_apply，然后再设置recovery_min_apply_delay = 1min，会发现生产库的事务会发生等待，直到备库过一分钟recovery结束后才完成，所以要避免这种情况发生。

7.3、如果把如果pg数据库的归档日志都存放在一个目录下，那么将来主从切换的时候会造成错误，导致启动失败。
```

八、提高主库的可用性和故障处理
处于同步复制的备用服务器发生故障并且不再能够返回ACK响应，主服务器仍将继续永远等待响应。因此，无法提交正在运行的事务，也无法启动后续查询处理。流式复制不支持通过超时自动还原到异步模式的功能。

两种解决办法：
 使用多个备用服务器来提高系统可用性
 通过手动执行从同步模式切换到异步模式
（1） 将参数synchronous_standby_names设置为空字符串。
（2） 使用reload选项执行pg_ctl命令。
postgres> pg_ctl -D $PGDATA reload

```
我们讨论第一种解决办法：使用多个备用服务器来提高系统可用性。
```

1、 配置主库postgres.conf文件：
synchronous_standby_names = 'standby_pg2,standby_pg3'
--此时pg2的优先级比pg3的要高
2、查看流复制状态：
testdb=# SELECT application_name AS host, sync_priority, sync_state FROM pg_stat_replication;
host | sync_priority | sync_state
-------------+---------------+------------
standby_pg2 | 1 | sync
standby_pg3 | 2 | potential

2、 把standby_pg2数据库关闭，则standby_pg3就会变成sync，而生产库进行dml操作不受到影响，因为此时standby_pg3替代了standby_pg2，成为第一备库。
testdb=# SELECT application_name AS host, sync_priority, sync_state FROM pg_stat_replication;
host | sync_priority | sync_state
-------------+---------------+------------
standby_pg3 | 2 | sync

3、 如果此时把standby_pg3也关闭，则主库的ddl和dml操作就会处于等待状态，因为当前没有可用的备库来进行实时同步。
4、 接下来只要启动任一的备库，就会立刻成为第一备库，则生产库就能够继续进行数据操作。

注意：
根据故障类型的不同，通常可以在故障发生后立即检测到故障，而有时在故障发生和检测到故障之间可能有一个时间间隔。特别是，如果同步备用服务器中发生这一种类型的故障（硬件和网络的故障检测），则主服务器上的所有事务处理都将停止，直到检测到备用服务器的故障为止，即使多个潜在的备用服务器可能已在工作。
把pg数据库的日志功能打开，可以查看更多的信息：
postgres.conf添加参数如下：

```
log_destination = 'csvlog'         
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d'  
log_truncate_on_rotation = off
log_rotation_age = 1d               
log_rotation_size = 0   
log_error_verbosity = verbose 
log_statement = all
```

经过测试，发现把日志目录存放在$PGDATA/pg_log下，能够记录的内容很多，经过观察发现pg的很多自身的命令其实在数据库里面都转换成sql语句。
比如：
\l

日志信息如下：

```
2020-04-29 04:53:23.367 EDT,"postgres","testdb",4655,"[local]",5ea93fed.122f,2,"idle",2020-04-29 04:50:53 EDT,4/9,0,LOG,00000,"statement: SELECT d.datname as ""Name"",
       pg_catalog.pg_get_userbyid(d.datdba) as ""Owner"",
       pg_catalog.pg_encoding_to_char(d.encoding) as ""Encoding"",
       d.datcollate as ""Collate"",
       d.datctype as ""Ctype"",
       pg_catalog.array_to_string(d.datacl, E'\n') AS ""Access privileges""
FROM pg_catalog.pg_database d
ORDER BY 1;",,,,,,,,"exec_simple_query, postgres.c:1045","psql"
```



### 常见问题解决

```
如果报错：
pg_basebackup: error: could not connect to server: could not connect to server: No route to host  Is the server running on host "pg1" (192.168.18.211) and accepting TCP/IP connections on port 1922?

解决方法：发现是系统防火墙的问题：

# 查看防火墙状态

service iptables status  

# 停止防火墙

service iptables stop  

# 永久关闭防火墙

chkconfig iptables off
```