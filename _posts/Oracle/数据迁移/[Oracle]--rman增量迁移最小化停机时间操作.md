---
title: [Oracle]--rman增量迁移最小化停机时间操作
date: 2020-06-18
---



​	使用oracle自带工具rman先全备，在迁移当天仅利用剩余归档和增量备份实现最小化停机时间。



## [全备]



### 源库全备份

```
mkdir -p /u01/myrman

rman target /

run {
    allocate channel ch1 type disk;
    allocate channel ch2 type disk;
    allocate channel ch3 type disk;
    allocate channel ch4 type disk;
    sql 'alter system archive log current';
    sql 'alter system archive log current';
    backup format '/u01/myrman/db_full_%T_%s_%p' as compressed backupset incremental level 0 database plus archivelog delete all input;
    backup format '/u01/myrman/db_controlfile_%T_%s_%p' current controlfile;
    sql 'alter system archive log current';
    backup format '/u01/myrman/db_arch_%Y%M%D_%s_%p' archivelog all;
    release channel ch1;
    release channel ch2;
    release channel ch3;
    release channel ch4;
}

sql 'alter system archive log current';

--备份归档
backup format '/u01/myrman/db_arch_%T_%s_%p' archivelog all;
```

### 源库参数文件及备份集传至备库

```
create pfile='/u01/myrman/pfile20200616.ora' from spfile;

cd /u01/myrman/

scp pfile20200616.ora oracle@新库IP:/u01/myrman/

scp *  oracle@新库IP:/u01/myrman/
```

### 新库修改参数文件启动

```
create spfile from pfile='/u01/myrman/pfile20200616.ora';
```

### 新库识别备份集

```
startup nomount

restore controlfile from '/u01/myrman/db_full_20200523_9_1';

sql 'alter database mount';

catalog start with '/u01/myrman';
```



### 修改数据文件在新库的存储位置

<span style="color:red">注意：如果新库文件位置与源库一致，可以不用修改，跳过此步骤。</span>

```
set lines 150
col tname for a10
col dname for a65
select t.ts#,t.name tname,d.file#,d.name dname,d.status from v$tablespace t,v$datafile d where t.ts#=d.ts#;

--对数据文件重命名查询语句
select 'set newname for datafile '||d.file#||' to '''||d.name||''';' from v$datafile d,v$tablespace t where d.ts#=t.ts# and t.INCLUDED_IN_DATABASE_BACKUP='YES';
```

### 新库第一次全备恢复

```
run{
set newname for datafile 1 to 'D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSTEM01.DBF';
set newname for datafile 2 to 'D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSAUX01.DBF';
set newname for datafile 3 to 'D:\APP\ADMINISTRATOR\ORADATA\ORCL\UNDOTBS01.DBF';
set newname for datafile 4 to 'D:\APP\ADMINISTRATOR\ORADATA\ORCL\USERS01.DBF';
........等等
restore database;
switch datafile all;
}
```

### 查看新库日志文件信息

如果与预期不一致，需要手动配置、改名或增加日志文件。

```
select 'alter database rename file '''||member||''' to '''||member||''';' from v$logfile;

--改名
alter database rename file '/u01/oradata/prod/redo03.log' to '/u01/oradata/prod/redo03.log';
alter database rename file '/u01/oradata/prod/redo02.log' to '/u01/oradata/prod/redo02.log';
alter database rename file '/u01/oradata/prod/redo01.log' to '/u01/oradata/prod/redo01.log';

--删除
alter database drop logfile group 2;

--增加
alter database add logfile group 3 ('/u01/oradata/prod/redo03.log') size 100M;
```



## [增量]

停止监听禁止会话连入，重启数据库与mount态，备份增量数据。

```
lsnrctl stop

archive log list;

alter system switch logfile;

alter system switch logfile;

shutdown immediate

startup mount;

BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 1 DATABASE FORMAT '/u01/myrman/two/db_incr_%T_%s_%p';

backup current controlfile format '/u01/myrman/two/ctlfile_%T_%s_%p';
```



### 备库恢复增量数据

```
--备库
mkdir -p /u01/myrman/two

scp * oracle@节点二IP:/u01/myrman/two/

shutdown immediate;

startup nomount;

restore controlfile from '/u01/myrman/two/ctlfile_20200523_18_1';

sql 'alter database mount';

catalog start with '/u01/myrman/two';

alter database mount;

restore database;

--noredo在rman恢复数据库时，不使用redo日志前滚数据。
recover database noredo;

alter database open resetlogs;
```



### 重建临时表空间

如果临时表空间路径与源库不一致，可以重建临时文件。

```
set linesize 120
col name for a50
select FILE#,BYTES/1024/1024/1024,NAME from v$tempfile;

alter database tempfile '/u01/oradata/prod/TEMP01.DBF' drop;

ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/oradata/prod/TEMP01.DBF' SIZE 10G autoextend on
```