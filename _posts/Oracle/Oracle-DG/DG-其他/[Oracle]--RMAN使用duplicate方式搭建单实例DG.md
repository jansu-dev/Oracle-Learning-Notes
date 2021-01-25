---
title:[Oracle]--RMAN使用duplicate方式搭建单实例DG
date:2020-07-23
---



RMAN使用duplicate方式搭建DG

- [主库操作](主库操作)
- [备库操作](备库操作)



### 主库操作

修改数据库为强制记日志，这是必须的操作，主库的每一步操作都得记录到日志中去。
alter database force logging;

---rman登陆，备份主库

```
run{
allocate channel c1 type disk;
allocate channel c2 type disk;
sql 'alter system archive log current';
backup format '/u01/myrman/db_%U_%T' skip inaccessible filesperset 5 database;
sql 'alter system archive log current';
backup format '/u01/myrman/db_%U_%T' skip inaccessible filesperset 5 archivelog all;
backup current controlfile for standby format='/u01/myrman/control_%U';
release channel c2;
release channel c1;
}
```

传送至备库

```
scp /u01/myrman/* 192.168.169.201:/u01/myrman/
```



### 备库操作

```
create spfile from pfile=/home/oracle/initstd.ora

create spfile from pfile;

startup force nomount;
```

任选一端

```
rman target sys/oracle@uni_dg1 auxiliary sys/oracle@uni_dg2 nocatalog
或者
rman target sys/oracle@uni_dg1 auxiliary sys/oracle@192.168.169.201:1521/prodstd nocatalog
```

复制数据

```
duplicate target database for standby nofilenamecheck dorecover;
```

查看出现standby字样

```
select group#,type,member from v$logfile;  

 ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 ('/u01/app/oracle/oradata/orcl/redo04.log') size 50M; 

ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 ('/u01/app/oracle/oradata/orcl/redo05.log') size 50M; 

ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 ('/u01/app/oracle/oradata/orcl/redo06.log') size 50M; 

ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 ('/u01/app/oracle/oradata/orcl/redo07.log') size 50M;

alter database recover managed standby database disconnect from session using current logfile;
```

--如果出现报错可以使用一下语句

```
alter database recover managed standby database cancel;

alter system set standby_file_management=MANUAL;
```