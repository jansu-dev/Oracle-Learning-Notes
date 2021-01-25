---
title:DG常用语句
date:2020-07-07
---



##### DG常用语句:

- [查看主备库基本信息](查看主备库基本信息)

- [检查主备库归档日志同步情况](检查主备库归档日志同步情况)

- [应用归档（mrp进程）](应用归档（mrp进程）)

- [取消应用归档（mrp进程）](取消应用归档（mrp进程）)

- [备库查看gap](备库查看gap)

- [主备切换语句](主备切换语句)

- [检查主备两边的序号](检查主备两边的序号)

- [检查备库是否开启实时应用](检查备库是否开启实时应用)

- [启动mrp进程时查看日志](启动mrp进程时查看日志)

- [验证备库状态](验证备库状态)






### 查看主备库基本信息

```
select open_mode,protection_mode,database_role,switchover_status from v$database;
```



### 检查主备库归档日志同步情况

```
select thread#,sequence#,name,standby_dest,archived,applied,status from v$archived_log order by 1,2;
```



### 应用归档（mrp进程）

```
alter database recover managed standby database using current logfile disconnect from session;
```



### 取消应用归档（mrp进程）

```
alter database recover managed standby database cancel;
```



### 备库查看gap

```
select * from v$archive_gap;
```



### 主备切换语句

```
alter database commit to switchover to physical standby with session shutdown;
```



### 检查主备两边的序号

```
select max(sequence#) from v$log;   
```



### 检查备库是否开启实时应用

```
select recovery_mode from v$archive_dest_status where dest_id=2;
```



### 启动mrp进程时查看日志

```
alter database recover managed standby database using current logfile disconnect;

alter database recover managed standby database disconnect from session;  --后台执行

alter database recover managed standby database --前台执行，执行这个可以看到报错的情况

如果有报错，查看alert日志和log.xml日志
```



### 验证备库状态

```
select process,status,sequence# from v$managed_standby;
```