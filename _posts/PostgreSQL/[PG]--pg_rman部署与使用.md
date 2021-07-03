---
title:[PG]--pg_rman部署与使用
date:2020-09-17
---



​		数据库的管理首要保证数据不丢失，日常备份备份必不可少。pg_rman部署与使用基于PG块结构进行备份的组件，本篇文章简要介绍的pg_rman部署与使用。

- [插件下载](插件下载)

- [插件部署](插件部署)

- [备份操作](备份操作)

- [恢复操作](恢复操作)

  

### 插件下载

> [**官网镜像下载**](https://github.com/ossc-db/pg_rman/releases)
> [**JanNest镜像下载**](https://pan.baidu.com/s/1iKQvocWkj9_YBscYXMvYjg) 提取码：6tqv

<span style='color:red'>**注意**</span>

<span style='color:red'>1. 下载对应数据库版本及操作系统的pg_rman源码；</span>
<span style='color:red'>2. 源码是与操作系统无关的，在编译安装的时自动适配操作系统；</span>



### 插件部署

上传安装包并解压安装（postgres用户安装）

```
# tar vxf pg_rman-1.3.9-pg12.tar.gz
# cd pg_rman-1.3.9-pg12
# make
# make install
```

开启归档模式

```
vi /usr/local/pg12.2/data/postgresql.conf

archive_mode = on
archive_command = 'cp %p /home/postgres/arch/%f'
```

初始化环境
--设置备份目录，在用户配置文件里面添加如下：

```
export BACKUP_PATH=/home/postgres/pg_rman_bk
```

--初始化备份目录，验证归档路径，日志目录，同时在备份路径下产生跟目标数据库相关的文件。

```
$ pg_rman init
pg_rman init
INFO: ARCLOG_PATH is set to '/home/postgres/arch'
INFO: SRVLOG_PATH is set to '/usr/local/pg12.2/data/pg_log'
```

--查看备份路径下的内容：

```
ls pg_rman_bk1/
20200603  backup  pg_rman.ini  system_identifier  timeline_history
```



### 备份操作

--对数据库做全备：

```
$ pg_rman backup --backup-mode=full -B /home/postgres/pg_rman_bk/ -C -P
```

--验证数据库备份（必须要验证，否则后续无法做增量备份）：

```
$ pg_rman validate
INFO: validate: "2020-06-03 22:13:52" backup and archive log files by CRC
INFO: backup "2020-06-03 22:13:52" is valid
```

--查看备份信息：

```
$ pg_rman show
=====================================================================
 StartTime           EndTime              Mode    Size   TLI  Status 
=====================================================================
2020-06-03 22:13:52  2020-06-03 22:13:54  FULL    15MB     1  OK
```

--对数据库进行增量备份（对数据库做增量备份前先做full备份）：

```
$ pg_rman backup --backup-mode=incremental 
INFO: copying database files
INFO: copying archived WAL files
INFO: backup complete
```

再做一次增量备份：

```
$ pg_rman backup --backup-mode=incremental 
```

--查看备份信息：

```
$ pg_rman show

2020-06-03 22:23:27  2020-06-03 22:23:30  INCR    67MB     1  DONE
2020-06-03 22:21:39  2020-06-03 22:21:42  INCR    33MB     1  DONE
2020-06-03 22:21:15  2020-06-03 22:21:18  INCR    83MB     1  OK
观察每次备份目录下的内容，归档日志如果没有删除的话，每次都会备份。
```

--清除归档，清除指定归档之前的归档日志：

```
pg_archivecleanup /home/postgres/arch_1 000000010000000000000017
```

--可以在postgresql.conf文件中添加自动清除归档的命令：

```
archive_cleanup_command = 'pg_archivecleanup  /home/postgres/arch_1 %r'
```

--删除备份：

```
$ pg_rman show
=====================================================================
 StartTime           EndTime              Mode    Size   TLI  Status 
=====================================================================
2020-06-03 22:39:20  2020-06-03 22:39:22  INCR    67MB     1  DONE
2020-06-03 22:24:09  2020-06-03 22:24:12  INCR   100MB     1  DONE
2020-06-03 22:23:27  2020-06-03 22:23:30  INCR    67MB     1  DONE
2020-06-03 22:21:39  2020-06-03 22:21:42  INCR    33MB     1  DONE
2020-06-03 22:21:15  2020-06-03 22:21:18  INCR    83MB     1  OK
2020-06-03 22:13:52  2020-06-03 22:13:54  FULL    15MB     1  OK

$ pg_rman delete "2020-06-03 22:39:20" 
INFO: delete the backup with start time: "2020-06-03 22:39:20"
INFO: delete the backup with start time: "2020-06-03 22:24:09"
INFO: delete the backup with start time: "2020-06-03 22:23:27"
INFO: delete the backup with start time: "2020-06-03 22:21:39"
WARNING: cannot delete backup with start time "2020-06-03 22:21:15"
DETAIL: This is the incremental backup necessary for successful recovery.
WARNING: cannot delete backup with start time "2020-06-03 22:13:52"
DETAIL: This is the latest full backup necessary for successful recovery.
实验发现，删除这个时间点相关联的增量备份，除了第一次增量备份和full备份
```

--强制删除指定时间点之前的所有备份：

```
pg_rman delete "2020-06-03 22:21:15" -f
```



### 恢复操作

恢复
1、如果当前使用的redolog损坏，只能做不完全恢复，系统自动判断：
--恢复数据库集群：

```
pg_rman restore
```

2、启动数据库：

```
pg_ctl start
```

3、指定恢复到某个时间点：

```
pg_rman retore --recovery-target-time="2020-06-03 23:42:26"
```

4、修改恢复文件

```
[postgres@localhost /]$ cd $PG_HOME/data/
[postgres@localhost data]$ ll |grep  recovery 
recovery.signal
[postgres@localhost data]$ mv recovery.signal recovery.done


[postgres@localhost data]$ pg_ctl restart -m fast
waiting for server to shut down....2020-09-17 13:17:36.616 EDT [53949] LOG:  received fast shutdown request
......
......
2020-09-17 13:17:36.761 EDT [54015] LOG:  database system is ready to accept connections
 done
server started
```