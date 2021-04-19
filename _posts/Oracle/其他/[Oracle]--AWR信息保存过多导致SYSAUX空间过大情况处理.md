---
title:[Oracle]--AWR信息保存过多导致SYSAUX空间过大情况处理
date:2020-08-31
---



#### 目录

- ##### [查找空间占用高的原因](查找空间占用高的原因)

- ##### [调整AWR设置减少空间占用](调整AWR设置减少空间占用)

- ##### [效果对比](效果对比)

- ##### [参考文章](参考文章)







​		在oracle10g的时候，引入sysaux作为system表空间的辅助表空间，存放一些基本组件，如oem,streams等。 如不加管理，会出现持续占用物理磁盘空间的现象。

![案例.jpg](http://cdn.lifemini.cn/dbblog/20200821/4b97eab598e44cb795bdd8fd5828468a.jpg)



### 查找空间占用高的原因

查询最占空间的组件

```
col MoveProcedure for a30
set linesize 100
set linesize 1000

SELECT OCCUPANT_NAME "Item",
      SPACE_USAGE_KBYTES / 1048576 "SpaceUsed(GB)",
      SCHEMA_NAME "Schema",
      MOVE_PROCEDURE "MoveProcedure"
  FROM V$SYSAUX_OCCUPANTS
 WHERE SPACE_USAGE_KBYTES > 1048576
 ORDER BY "SpaceUsed(GB)" DESC; 


Item             SpaceUsed(GB)          Schema                MoveProcedure
-------------------- ------------- -------------------- ------------------------------
SM/AWR                13.7681274             SYS
```

如果OCCUPANT_NAME列值为SM/AWR（Server Manageability - Automatic Workload Repository），
那么表示AWR信息占用过大；
如果OCCUPANT_NAME列值为SM/OPTSTAT（ServerManageability - Optimizer Statistics History），
那么表示优化器统计信息占用过大。

如已判定是AWR的报告占用空间过高，
可使用如下命令查看AWR信息。

```
sqlplus / as sysdba @$ORACLE_HOME/rdbms/admin/awrinfo.sql
```



### 调整AWR设置减少空间占用

取前10依据占空间大小排序的段名

```
col SEGMENT_NAME for a30
col SIZE_M for a30
set linesize 1000

SELECT * from (
 SELECT D.SEGMENT_NAME, D.SEGMENT_TYPE,SUM(BYTES)/1024/1024/1024  SIZE_G
  FROM DBA_SEGMENTS D
 WHERE D.TABLESPACE_NAME = 'SYSAUX'
 GROUP BY D.SEGMENT_NAME, D.SEGMENT_TYPE
 ORDER BY SIZE_G DESC) where rownum <=10;
```

查询AWR保存时间

```
col SNAP_INTERVAL for a20
col RETENTION for a30
col TOPNSQL for a30
set linesize 1000

SELECT * FROM DBA_HIST_WR_CONTROL;

      DBID  SNAP_INTERVAL              RETENTION                    TOPNSQL
---------- -------------------- ------------------------------ ------------------------------
2730892750  +00000 01:00:00.0           +00008 00:00:00.0           DEFAULT
```

修改AWR-SNAPSHOT保存时间

```
EXEC DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(INTERVAL=>60,RETENTION=>8*24*60);
```

查询并删除时间片

```
SELECT MIN(SNAP_ID),MAX(SNAP_ID) FROM DBA_HIST_SNAPSHOT;

SELECT MIN(SNAP_ID),MAX(SNAP_ID) FROM DBA_HIST_ACTIVE_SESS_HISTORY;

select dbid from v$database;

  DBID
----------
2730892750

BEGIN
    DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE(
    LOW_SNAP_ID => 1,
    HIGH_SNAP_ID => 19591,
    DBID=> 2730892750);
END;
```

<span style='color:red'>**注意：**</span>

<span style='color:red'>1. DROP_SNAPSHOT_RANGE是delete操作，长时间删除可能导致进程挂死。</span>
<span style='color:red'>2. 建议优先执行下面语句，拼接删除语句truncate基表。</span>

```
col STATEMENTS for a50
set pagesize 1000
select distinct 'truncate  table '||segment_name||';' statements,s.bytes/1024/1024 sizeM
  from dba_segments s
 where s.segment_name like 'WRH$%'
   and segment_type in ('TABLE PARTITION', 'TABLE')
   and s.bytes/1024/1024>100
   order by s.bytes/1024/1024 desc;
```

1. 因为truncate操作会导致索引失效，需要重建相关索引。

```
col TABLE_OWNER for a20
col TABLE_NAME for a30         
col PARTITION_NAME for a30
set linesize 1000

select TABLE_OWNER,TABLE_NAME,PARTITION_NAME from dba_tab_partitions where TABLE_NAME='WRH$_ACTIVE_SESSION_HISTORY';

select 'ALTER TABLE WRH$_ACTIVE_SESSION_HISTORY MOVE PARTITION ' ||PARTITION_NAME||' UPDATE GLOBAL INDEXES;' from dba_tab_partitions where TABLE_NAME='WRH$_ACTIVE_SESSION_HISTORY';

select 'ALTER INDEX WRH$_ACTIVE_SESSION_HISTORY_PK REBUILD PARTITION ' ||PARTITION_NAME||';' from dba_tab_partitions where TABLE_NAME='WRH$_ACTIVE_SESSION_HISTORY';
```



### 效果对比

```
--操作前

TABLESPACE_NAME                 TOTAL_G     FREE_G     USED_G     USED_PERCENT
------------------------------ ---------- ---------- ---------- ------------
SYSAUX                            16.33        1.62      14.71        90.06

--操作后

TABLESPACE_NAME                 TOTAL_G     FREE_G     USED_G     USED_PERCENT
------------------------------ ---------- ---------- ---------- ------------
SYSAUX                            16.33      13.08       3.25        19.92
```



### 参考文章

- ##### [数据百科](https://www.baikedb.com/post-15.html)