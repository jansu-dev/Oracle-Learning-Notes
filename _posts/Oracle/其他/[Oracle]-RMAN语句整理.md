---
title:[Oracle]-RMAN语句整理
date:2020-11-06
---



#### RMAN语句整理

- ###### [rman备份16天以内的所有备份](rman备份16天以内的所有备份)

- ###### [查看所有备份集](查看所有备份集)

- ###### [查找某个备份集中包含数据文件](查找某个备份集中包含数据文件)

- ###### [查询某个备份集中控制文件](查询某个备份集中控制文件)

- ###### [查看某个备份集中归档日志](查看某个备份集中归档日志)

- ###### [查看某个备份集SPFILE](查看某个备份集SPFILE)

- ###### [rman配置信息](rman配置信息)

- ###### [获取DB_LINK的IP地址](获取DB_LINK的IP地址)

- ###### [删除几天之前压缩备份集](删除几天之前压缩备份集)

- ###### [检查并删除过期备份集](检查并删除过期备份集)

- ###### [查看所有归档备份文件](查看所有归档备份文件)

- ###### [清理归档日志的压缩备份文件](清理归档日志的压缩备份文件)

- ###### [清理归档日志的备份文件](清理归档日志的备份文件)

- ###### [清理归档日志](清理归档日志)









### RMAN语句整理

- ##### rman备份16天以内的所有备份

```
select to_char(start_time, 'yyyymmdd'),
       round(sum(a.output_bytes) / 1024 / 1024 / 1024, 2) gb
  from v$rman_status a
 where a.start_time between trunc(sysdate - 16) and trunc(sysdate)
 group by to_char(start_time, 'yyyymmdd')
 order by to_char(start_time, 'yyyymmdd');
```

- ##### 查看所有备份集

```
SELECT A.RECID "BACKUP SET",
       A.SET_STAMP,
       DECODE(B.INCREMENTAL_LEVEL,
              '',
              DECODE(BACKUP_TYPE, 'L', 'Archivelog', 'Full'),
              1,
              'Incr-1级',
              0,
              'Incr-0级',
              B.INCREMENTAL_LEVEL) "Type LV",
       B.CONTROLFILE_INCLUDED "包含CTL",
       DECODE(A.STATUS,
              'A',
              'AVAILABLE',
              'D',
              'DELETED',
              'X',
              'EXPIRED',
              'ERROR') "STATUS",
       A.DEVICE_TYPE "Device Type",
       A.START_TIME "Start Time",
       A.COMPLETION_TIME "Completion Time",
       A.ELAPSED_SECONDS "Elapsed Seconds",
       --a.BYTES/1024/1024/1024 "大小(G)",
       --a.COMPRESSED,
       A.TAG    "Tag",
       A.HANDLE "Path"
  FROM GV$BACKUP_PIECE A, GV$BACKUP_SET B
 WHERE A.SET_STAMP = B.SET_STAMP
   AND A.DELETED = 'NO'
   and a.set_count = b.set_count
 ORDER BY A.COMPLETION_TIME DESC;
```

- ##### 查找某个备份集中包含数据文件

```
SELECT distinct c.file#,
                A.SET_STAMP,
                D.NAME,
                C.CHECKPOINT_CHANGE#,
                C.CHECKPOINT_TIME
  FROM V$BACKUP_DATAFILE C, V$BACKUP_PIECE A, V$DATAFILE D
 WHERE A.SET_STAMP = C.SET_STAMP
   AND D.FILE# = C.FILE#
   AND A.DELETED = 'NO'
   AND c.set_stamp = &set_stamp
 ORDER BY C.FILE#;
```

- ##### 查询某个备份集中控制文件

```
SELECT DISTINCT A.SET_STAMP,
                D.NAME,
                C.CHECKPOINT_CHANGE#,
                C.CHECKPOINT_TIME
  FROM V$BACKUP_DATAFILE C, V$BACKUP_PIECE A, V$CONTROLFILE D
 WHERE A.SET_STAMP = C.SET_STAMP
   AND C.FILE# = 0
   AND A.DELETED = 'NO'
   AND C.SET_STAMP = &SET_STAMP;

```

- ##### 查看某个备份集中归档日志

```
SELECT DISTINCT B.SET_STAMP,
                B.THREAD#,
                B.SEQUENCE#,
                B.FIRST_TIME,
                B.FIRST_CHANGE#,
                B.NEXT_TIME,
                B.NEXT_CHANGE#
  FROM V$BACKUP_REDOLOG B, V$BACKUP_PIECE A
 WHERE A.SET_STAMP = B.SET_STAMP
   AND A.DELETED = 'NO'
   AND B.SET_STAMP = &SET_STAMP
 ORDER BY THREAD#, SEQUENCE#;
```

- ##### 查看某个备份集SPFILE

```
SELECT DISTINCT A.SET_STAMP, B.COMPLETION_TIME, HANDLE
  FROM V$BACKUP_SPFILE B, V$BACKUP_PIECE A
 WHERE A.SET_STAMP = B.SET_STAMP
   AND A.DELETED = 'NO'
   AND B.SET_STAMP = &SET_STAMP;
```

- ##### rman配置信息

```
SELECT NAME, VALUE FROM V$RMAN_CONFIGURATION;





































```

- ##### 获取DB_LINK的IP地址

```
select owner,db_link,username,regexp_substr(host,'(\d*[.]){3}\d*') from dba_db_links;
```

- ##### 删除几天之前压缩备份集

```
delete backup completed  before 'sysdate-&1';
```

- ##### 检查并删除过期备份集

```
crosscheck backup completed  before 'sysdate-&1' (or BETWEEN 'sysdate-&1'  AND 'sysdate-&2' )
&
delete expired backup completed  before 'sysdate-&1' (or BETWEEN 'sysdate-&1'  AND 'sysdate-&2' )
```

- ##### 查看所有归档备份文件

```
list backup of archivelog all;
```

- ##### 清理归档日志的压缩备份文件

```
delete backup of archivelog all completed before "sysdate-7";
```

- ##### 清理归档日志的备份文件

```
delete backup of archivelog until time‘sysdate-30;
```

- ##### 清理归档日志

```
delete noprompt archivelog until time‘sysdate-7;
```