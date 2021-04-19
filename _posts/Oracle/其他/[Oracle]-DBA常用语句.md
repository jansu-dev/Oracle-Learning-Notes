---
title:[Oracle]-DBA常用语句
date:2020-06-27
---





#### 目录

- ##### [坏块问题解决](坏块问题解决)

- ##### [外键查询](外键查询)

- ##### [物化视图相关](物化视图相关)

- ##### [会话相关](会话相关)

- ##### [SQL相关](SQL相关)

- ##### [空间相关](空间相关)

- ##### [DB_LINK使用](DB_LINK使用)

- ##### [统计信息相关](统计信息相关)

- ##### [UNDO相关](UNDO相关)

- ##### [归档相关](归档相关)

- ##### [锁相关](锁相关)

- ##### [日志相关](日志相关)

- ##### [密码与授权](密码与授权)

- ##### [AWR相关](AWR相关)

##### 















#### DBA常用语句

### 坏块问题解决

```sql
--确定发生坏块的数据库对象 
SELECT tablespace_name, segment_type, owner, segment_name FROM  dba_extents WHERE  file_id = <AFN> AND <BLOCK> between block_id AND block_id+blocks-1; 

--dbms包标记坏块
exec DBMS_REPAIR.SKIP_CORRUPT_BLOCKS('<schema>','<tablename>');

--标记后重重建表
create table corrupt_table_bak   as   select * from corrupt_table;
```



### 外键查询

查询无主键表

```
SELECT table_name  FROM all_tables  WHERE owner = USER  MINUS  SELECT table_name  FROM all_constraints  
WHERE owner = USER  AND constraint_type = 'P'
```

查询外键无索引的列

```
col column_name for a30;
col owner for a10;

SELECT c.owner, c.constraint_name, c.table_name, cc.column_name, c.status
  FROM dba_constraints c, dba_cons_columns cc
 WHERE c.constraint_type = 'R'
   AND c.owner NOT IN
       ('ADM_PARALLEL_EXECUTE_TASK','ANONYMOUS','APEX_030200','APEX_ADMINISTRATOR_ROLE','APEX_PUBLIC_USER',
        'APPQOSSYS','AQ_ADMINISTRATOR_ROLE','AQ_USER_ROLE','CONNECT','CSW_USR_ROLE','CTXAPP','CTXSYS',
        'CWM_USER','DATAPUMP_EXP_FULL_DATABASE','DATAPUMP_IMP_FULL_DATABASE','DBA','DBFS_ROLE','DBSNMP',
        'DELETE_CATALOG_ROLE','DIP','EXECUTE_CATALOG_ROLE','EXFSYS','EXP_FULL_DATABASE','FLOWS_FILES',
        'GATHER_SYSTEM_STATISTICS','HS_ADMIN_EXECUTE_ROLE','HS_ADMIN_ROLE','HS_ADMIN_SELECT_ROLE',
        'IMP_FULL_DATABASE','JAVADEBUGPRIV','JAVASYSPRIV','LOGSTDBY_ADMINISTRATOR','MDDATA','MDSYS',
        'MGMT_USER','MGMT_VIEW','OEM_ADVISOR','OEM_MONITOR','OLAPSYS','OLAP_DBA','OLAP_USER','OLAP_XS_ADMIN',
        'ORACLE_OCM','ORDADMIN','ORDDATA','ORDPLUGINS','ORDSYS','OUTLN','OWB$CLIENT','OWBSYS','OWBSYS_AUDIT',
        'PUBLIC','RECOVERY_CATALOG_OWNER','RESOURCE','SCHEDULER_ADMIN','SCOTT','SELECT_CATALOG_ROLE',
        'SI_INFORMTN_SCHEMA','SPATIAL_CSW_ADMIN','SPATIAL_CSW_ADMIN_USR','SPATIAL_WFS_ADMIN',
        'SPATIAL_WFS_ADMIN_USR','SQLTXADMIN','SQLTXPLAIN','SQLT_USER_ROLE','SYS','SYSMAN','SYSTEM',
        'WFS_USR_ROLE','WMSYS','WM_ADMIN_ROLE','XDB','XDBADMIN','PERFSTAT','GSMADMIN_INTERNAL')
   AND c.owner = cc.owner
   AND c.constraint_name = cc.constraint_name
   AND NOT EXISTS
 (SELECT 'x'
          FROM dba_ind_columns ic
         WHERE cc.owner = ic.table_owner
           AND cc.table_name = ic.table_name
           AND cc.column_name = ic.column_name
           AND cc.position = ic.column_position
           AND NOT EXISTS
         (SELECT owner, index_name
                  FROM dba_indexes i
                 WHERE i.table_owner = c.owner
                   AND i.index_Name = ic.index_name
                   AND i.owner = ic.index_owner
                   AND (i.status = 'UNUSABLE' OR
                       i.partitioned = 'YES' AND EXISTS
                        (SELECT 'x'
                           FROM dba_ind_partitions ip
                          WHERE status = 'UNUSABLE'
                            AND ip. index_owner = i. owner
                            AND ip. index_Name = i. index_name
                         UNION ALL
                         SELECT 'x'
                           FROM dba_ind_subpartitions isp
                          WHERE status = 'UNUSABLE'
                            AND isp. index_owner = i. owner
                            AND isp. index_Name = i. index_name))))
 ORDER BY 1, 2;
```



### 物化视图相关

判断物化视图是如何刷新

```
select mview_name, refresh_mode, refresh_method,
last_refresh_type, last_refresh_date
from user_mviews
/
```



### 会话相关

查看活动会话

```
select INST_ID,count(*) from Gv$session t where t.STATUS='ACTIVE' group by t.INST_ID;
```

查看数据库的等待事件信息

```
select status, event, count(*) x
  from Gv$session
 where username is not null
 group by status, event
 order by x desc;
```

操作系统杀掉所有连接的会话

```
ps -ef | grep oracle | grep LOCAL=NO |cut -c 9-15 |xargs  kill -9
```



### SQL相关

抓慢SQL

```
select /*+parallel(6)*/sa.sql_text,ss.username,ss.module,ss.last_call_et
  from v$process pr, v$session ss, v$sqlarea sa
 where pr.addr = ss.paddr and ss.sql_hash_value=sa.hash_value
   and ss.username is not null
   and status = 'ACTIVE'
   and ss.module <> 'DBMS_SCHEDULER'
 order by ss.last_call_et desc, ss.sid;
```

通过进程ID查sql

```
select sa.* from v$process pr,v$session ss,v$sqlarea sa 
where pr.addr=ss.paddr and ss.sql_hash_value=sa.hash_value 
and pr.spid=27409
```



### 集群相关

root用户下RAC启停集群

```
crsctl stop crs 停集群
crsctl start crs 启动集群
```

grid用户查看进程状态，出现问题时重启集群资源

```
crsctl stat res -t -init 查看所有进程状态
crsctl start res ora.crsd -init  启动offline的进程  
crs_stat -t -v
```

rac关闭drm

```
alter system set "_gc_policy_time"=0 scope=spfile sid='*';
```

卸载grid

```
cd $GRID_HOME/deinstall
 ./deinstall -local
```

擦除asm信息

```
dd if=/dev/zero of=/dev/sdc bs=1024 count=512
```

重新注册

```
cd $GRID_HOME/crs/install/
./roothas.pl -deconfig -force -verbose

cd $GRID_HOME
./root.sh
```



### 空间相关

封装查询空间的包 dbms_space,查看对象利用情况

```sql
CREATE OR REPLACE PROCEDURE show_space(p_segname in varchar2,
                                       p_type    in varchar2 default 'TABLE',
                                       p_owner   in varchar2 default user) AS
  v_segname            varchar2(100);
  v_type               varchar2(10);
  l_free_blks          number;
  l_total_blocks       number;
  l_total_bytes        number;
  l_unused_blocks      number;
  l_unused_bytes       number;
  l_LastUsedExtFileId  number;
  l_LastUsedExtBlockId number;
  l_LAST_USED_BLOCK    number;
  PROCEDURE p(p_label in varchar2, p_num in number) IS
  BEGIN
    dbms_output.put_line(rpad(p_label, 40, '.') || p_num);
  END;
BEGIN
  v_segname := upper(p_segname);
  v_type    := p_type;
  if (p_type = 'i' or p_type = 'I') then
    v_type := 'INDEX';
  end if;
  if (p_type = 't' or p_type = 'T') then
    v_type := 'TABLE';
  end if;
  if (p_type = 'c' or p_type = 'C') then
    v_type := 'CLUSTER';
  end if;
  --以下部分不能用于 ASSM  
  dbms_space.free_blocks(segment_owner     => p_owner,
                         segment_name      => v_segname,
                         segment_type      => v_type,
                         freelist_group_id => 0,
                         free_blks         => l_free_blks);
  --以上部分不能用于 ASSM  
  dbms_space.unused_space(segment_owner             => p_owner,
                          segment_name              => v_segname,
                          segment_type              => v_type,
                          total_blocks              => l_total_blocks,
                          total_bytes               => l_total_bytes,
                          unused_blocks             => l_unused_blocks,
                          unused_bytes              => l_unused_bytes,
                          LAST_USED_EXTENT_FILE_ID  => l_LastUsedExtFileId,
                          LAST_USED_EXTENT_BLOCK_ID => l_LastUsedExtBlockId,
                          LAST_USED_BLOCK           => l_LAST_USED_BLOCK);
  --显示结果  
  p('Free Blocks', l_free_blks);
  p('Total Blocks', l_total_blocks);
  p('Total Bytes', l_total_bytes);
  p('Unused Blocks', l_unused_blocks);
  p('Unused Bytes', l_unused_bytes);
  p('Last Used Ext FileId', l_LastUsedExtFileId);
  p('Last Used Ext BlockId', l_LastUsedExtBlockId);
  p('Last Used Block', l_LAST_USED_BLOCK);
END;


--执行结果将如下所示 
set serveroutput on;  
exec show_space('test');
```

检查表空间碎片

```
select tablespace_name,count(*) chunks, max(bytes/1024/1024) max_chunk from dba_free_space group by tablespace_name;

alter tablespace 表空间名 coalesce;
```

查询碎片较多的索引

```
SELECT 'alter index ' || index_name || ' rebuild '  ||'tablespace INDEXES storage(initial 256K next 256K pctincrease 0);'  FROM all_indexes  WHERE ( tablespace_name != 'INDEXES'  OR next_extent != ( 256 * 1024 )  )  AND owner = USER
```

某表空间下段大小排序
`select * from ( select t.owner,t.segment_name ,sum(t.BYTES)/1024/1024/1024 a from dba_segments t where t.tablespace_name='TMS' group by t.owner,t.segment_name order by a desc) where rownum<=100;`

查看磁盘组命令

```
col name for a10
 select group_number,name,total_mb,free_mb,state,type from v$asm_diskgroup;

 col name for a10
 select group_number,name,total_mb/1024 total_G,free_mb/1024 free_G,state,type from v$asm_diskgroup;
```

查看表空间大小

```
col file_name for a10
col tablespace_name for a30
set linesize 120
set pagesize 1000

select f.tablespace_name tablespace_name,
       round((d.sumbytes / 1024 / 1024 / 1024), 2) total_g,
       round(f.sumbytes / 1024 / 1024 / 1024, 2) free_g,
       round((d.sumbytes - f.sumbytes) / 1024 / 1024 / 1024, 2) used_g,
       round((d.sumbytes - f.sumbytes) * 100 / d.sumbytes, 2) used_percent
  from (select tablespace_name, sum(bytes) sumbytes
          from dba_free_space
         group by tablespace_name) f,
       (select tablespace_name, sum(bytes) sumbytes
          from dba_data_files
         group by tablespace_name) d
 where f.tablespace_name = d.tablespace_name
 order by used_percent desc;
```

查看前10大小数据段

```
select * from (select rownum rn,tt.* from (select t.segment_name ,sum(t.BYTES)/1024/1024/1024 a from dba_segments t group by t.segment_name order by a desc) tt) ttt where ttt.rn<=10;
```

表空间增量

```
set pagesize 1000
set linesize 1000
col TABLESPACE_ID for a5
col name for a15
col INC_RATEX for a8

SELECT A.NAME, B.TABLESPACE_ID,B.DATETIME,B.USED_SIZE_MB,B.INC_MB,  
       CASE WHEN SUBSTR(INC_RATE,1,1)='.' THEN '0'||INC_RATE  
            WHEN SUBSTR(INC_RATE,1,2)='-.' THEN '-0'||SUBSTR(INC_RATE,2,LENGTH(INC_RATE))    
            ELSE INC_RATE  
       END AS INC_RATEX  
  FROM V$TABLESPACE A,   
       (  
           SELECT TABLESPACE_ID,DATETIME,  
                  USED_SIZE_MB,    
                  (DECODE(PREV_USE_MB,0,0,USED_SIZE_MB)-PREV_USE_MB) AS  INC_MB,   
                  TO_CHAR(ROUND((DECODE(PREV_USE_MB,0,0,USED_SIZE_MB)-PREV_USE_MB)/DECODE(PREV_USE_MB,0,1,PREV_USE_MB)*100,2))||'%' AS INC_RATE  
         FROM  
         (  
           SELECT TABLESPACE_ID,   
                  TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss')) DATETIME,   
                  MAX(TABLESPACE_USEDSIZE * 8 / 1024) USED_SIZE_MB,  
                  LAG(MAX(TABLESPACE_USEDSIZE * 8 / 1024),1,0) OVER(PARTITION BY TABLESPACE_ID ORDER BY TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss')) ) AS PREV_USE_MB  
             FROM DBA_HIST_TBSPC_SPACE_USAGE  
            WHERE TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss')) > TRUNC(SYSDATE - 30)  
            GROUP BY TABLESPACE_ID, TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss'))  )   
       ) B  
 WHERE A.TS# = B.TABLESPACE_ID  
 ORDER BY B.TABLESPACE_ID,DATETIME;
```

查看临时表空间使用
M

```
SELECT a.tablespace_name,
       a.BYTES / 1024 / 1024 total,
       a.bytes / 1024 / 1024 - nvl(b.bytes / 1024 / 1024, 0) freem
  FROM (SELECT tablespace_name, SUM(bytes) bytes
          FROM dba_temp_files
         GROUP BY tablespace_name) a,
       (SELECT tablespace_name, SUM(bytes_cached) bytes
          FROM v$temp_extent_pool
         GROUP BY tablespace_name) b
 WHERE a.tablespace_name = b.tablespace_name(+);
```

G

```
SELECT a.tablespace_name,
       a.BYTES / 1024 / 1024 / 1024 total,
       a.bytes / 1024 / 1024 / 1024 - nvl(b.bytes / 1024 / 1024 / 1024, 0) freem
  FROM (SELECT tablespace_name, SUM(bytes) bytes
          FROM dba_temp_files
         GROUP BY tablespace_name) a,
       (SELECT tablespace_name, SUM(bytes_cached) bytes
          FROM v$temp_extent_pool
         GROUP BY tablespace_name) b
 WHERE a.tablespace_name = b.tablespace_name(+);
```

查看当前时刻临时表空间的SQL占用情况

```
SELECT S.sid || ',' || S.serial# sid_serial,
       S.username,
       T.blocks * TBS.block_size / 1024 / 1024 /1024 GB_used,
       T.tablespace,
       T.sqladdr address,
       Q.hash_value,
       Q.sql_text
FROM v$sort_usage T, v$session S, v$sqlarea Q, dba_tablespaces TBS
WHERE T.session_addr = S.saddr
   AND T.sqladdr = Q.address(+)
   AND T.tablespace = TBS.tablespace_name
ORDER BY S.sid;
```

获取磁盘某目录下10的文件

```
du -h /u01/ | sort -rh | head -20
```



### DB_LINK使用

创建DB_LINK

```
CREATE database link db68
CONNECT TO cp_tms IDENTIFIED BY "password"
USING '(DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.2.196.68)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = tmsdb)
    )
  )';
```

删除db_link

```
drop database link db68;
```



### 统计信息相关

收集表统计信息

```
begin
dbms_stats.gather_table_stats(ownname => 'CP_TMS',
                              tabname => 'TMS_MAIL_MONITOR_2',
                              cascade => TRUE,
                              estimate_percent => dbms_stats.auto_sample_size,
                              method_opt => 'for all columns size auto',
                              degree =>8);
end;
```

收集用户统计信息

```
dbms_stats.gather_schema_stats(OWNNAME=>'SEC',ESTIMATE_PERCENT=>10,
DEGREE=>4,
cascade=>true);
```



### UNDO相关

查看当前UNDO表空间大小

```
select 
((select (nvl(sum(bytes),0)) 
from dba_undo_extents 
where tablespace_name = 'UNDOTBS1'
and status in ('ACTIVE','UNEXPIRED')) *100) / 
(select sum(bytes) 
from dba_data_files 
where tablespace_name = 'UNDOTBS1') "PCT_INUSE" 
from dual; 


select tablespace_name, status, sum(bytes/1024/1024) "MB"
from dba_undo_extents
group by tablespace_name, status
order by 1, 2;
```

过去7*24小时中UNDO表空间的平均使用量

```
select ur undo_retention,
       dbs db_block_size,
       ((ur * (ups * dbs)) + (dbs * 24)) / 1024 / 1024 as "M_bytes"
  from (select value as ur from v$parameter where name = 'undo_retention'),
       (select (sum(undoblks) / sum(((end_time - begin_time) * 86400))) ups
          from v$undostat),
       (select value as dbs from v$parameter where name = 'db_block_size');
```

按峰值情况计算UNDO表空间所需空间

```
select ur undo_retention,
       dbs db_block_size,
       ((ur * (ups * dbs)) + (dbs * 24)) / 1024 / 1024 as "M_bytes"
  from (select value as ur from v$parameter where name = 'undo_retention'),
       (select (undoblks / ((end_time - begin_time) * 86400)) ups
          from v$undostat
         where undoblks in (select max(undoblks) from v$undostat)),
       (select value as dbs from v$parameter where name = 'db_block_size');
```



### 归档相关

查看近段时间生产的归档量

```
select trunc(completion_time), sum(mb) / 1024 day_gb
  from (select name, completion_time, blocks * block_size / 1024 / 1024 mb
          from v$archived_log)
 group by trunc(completion_time)
 order by (trunc(completion_time)) desc ;
```

查看现有归档占用的空间

```
select sum(a.BLOCK_SIZE*a.BLOCKS)/1024/1024/1024 from v$archived_log a where a.DELETED='NO';
```

rac修改归档（DG）

```sql
--备份语句
create pfile='/home/oracle/chg_arch_before' from spfile;

--增加目录
alter diskgroup fradg add directory '+fradg/racdb/arch';

--主库切换路径
sqlplus / as sysdba

alter system set log_archive_dest_1='LOCATION=+fradg/racdb/arch VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=racdb';
```



### 锁相关

lock锁相关表

```
SELECT * FROM v$lock;
SELECT * FROM v$sqlarea;
SELECT * FROM v$session;
SELECT * FROM v$process ;
SELECT * FROM v$locked_object;
SELECT * FROM all_objects;
SELECT * FROM v$session_wait;
```

查看被锁的表

```
select b.owner,b.object_name,a.session_id,a.locked_mode from v$locked_object a,dba_objects b where b.object_id = a.object_id;
```

查看哪个用户哪个进程造成死锁

```
select b.username,b.sid,b.serial#,logon_time from v$locked_object a,v$session b where a.session_id = b.sid order by b.logon_time;
```

查看连接的进程

```
SELECT sid, serial#, username, osuser FROM v$session;
```

查出锁定表的sid, serial#,os_user_name, machine_name, terminal，锁的type,mode

```
SELECT s.sid, s.serial#, s.username, s.schemaname,
s.process, s.machine,
s.terminal, s.logon_time, l.type
FROM v$session s, v$lock l
WHERE s.sid = l.sid
AND s.username IS NOT NULL
ORDER BY sid;
```

kill被锁对象

```
SELECT S.SID, S.MACHINE, O.OBJECT_NAME, L.ORACLE_USERNAME, L.LOCKED_MODE, S.OSUSESR,
 'ALTER SYSTEM KILL SESSION '''|| S.SID || ', '|| S.SERIAL#||''';' AS KILL_COMMAND
 FROM V$LOCKED_OBJECT L, V$SESSION S, ALL_OBJECTS O
 WHERE L.SESSION_ID=S.SID AND L.OBJECT_ID=O.OBJECT_ID
```

关闭某用户所有连接

```
select 'ALTER SYSTEM KILL SESSION '''|| S.SID || ', '|| S.SERIAL#||''';' from v$session s where username ='用户名';
```



### 日志相关

查看日志切换时间

```
select sequence#,first_time,nexttime,round(((first_time-nexttime)*24)*60,2) diff 
from ( 
select sequence#,first_time, lag(first_time) over(order by sequence#) nexttime 
from v$log_history 
where thread#=1 
) order by sequence# desc; 
```

查看一个月内日志每天每小时日志切换情况

```
SELECT SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH:MI:SS'),1,5) Day,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'00',1,0)) H00,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'01',1,0)) H01, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'02',1,0)) H02,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'03',1,0)) H03,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'04',1,0)) H04,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'05',1,0)) H05,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'06',1,0)) H06,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'07',1,0)) H07,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'08',1,0)) H08,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'09',1,0)) H09,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'10',1,0)) H10,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'11',1,0)) H11, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'12',1,0)) H12,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'13',1,0)) H13, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'14',1,0)) H14,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'15',1,0)) H15, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'16',1,0)) H16, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'17',1,0)) H17, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'18',1,0)) H18, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'19',1,0)) H19, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'20',1,0)) H20, 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'21',1,0)) H21,
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'22',1,0)) H22 , 
SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'23',1,0)) H23, 
COUNT(*) TOTAL FROM v$log_history  a  GROUP BY SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH:MI:SS'),1,5) 
ORDER BY SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH:MI:SS'),1,5);
```



### 密码与授权

-------修改密码---------

```
--查看当前open用户 

select username,account_status,expiry_date,profile from dba_users; 

--查看目前的密码过期策略 

select * from dba_profiles s where s.profile='DEFAULT' and resource_name='PASSWORD_LIFE_TIME'; 

--修改密码过期策略 

alter profile default limit password_life_time unlimited;
```

-------查看角色---------

```
当前用户被激活的全部角色
select * from session_roles;

当前当前用户被授予的角色
select * from user_role_privs;

全部用户被授予的角色
select * from dba_role_privs;

查看某个用户所拥有的角色
select * from dba_role_privs where grantee='用户名';

查看某个角色所拥有的权限
select * from dba_sys_privs where grantee='CONNECT';

查看所有角色
select * from dba_roles;

当前用户所拥有的全部权限
select * from session_privs;   

 当前用户的系统权限
select * from user_sys_privs; 

当前用户的对象权限
select * from user_tab_privs;  

查询某个用户所拥有的系统权限
select * from dba_sys_privs ;  

查看角色(只能查看登陆用户拥有的角色)所包含的权限
select * from role_sys_privs;  


查看用户的系统权限(直接赋值给用户或角色的系统权限)
select * from dba_sys_privs;
select * from user_sys_privs;

-------查看基本权限---------

查看用户的对象权限：
select * from dba_tab_privs;
select * from all_tab_privs;
select * from user_tab_privs;


查看哪些用户有sysdba或sysoper系统权限(查询时需要相应权限)
select * from v$pwfile_users;


查看一个用户的所有系统权限(包含角色的系统权限)
select privilege from dba_sys_privs where grantee='SCOTT'  
union  
select privilege from dba_sys_privs where grantee in (select granted_role from dba_role_privs where grantee='SCOTT' ); 


查询当前用户可以访问的所有数据字典视图。 
select * from dict where comments like '%grant%';   

-------查询角色包括的权限---------


角色包含的系统权限   
select * from dba_sys_privs where grantee='角色名'  
select * from dba_sya_privs where grantee='COONNECT'; 
select * from role_sys_privs where role='角色名'   

角色包含的对象权限
select * from dba_tab_privs where grantee='角色名';    

有多少种角色;
select * from dba_roles;   

查看用户具有哪些角色
select * from dba_role_privs where grantee='用户名';    

查看哪些用户具有DBA的角色
select grantee from dba_role_privs where granted_role='DBA';
```



### AWR相关

awr及ash报告采集命令

```
@$ORACLE_HOME/rdbms/admin/ashrpt.sql
@$ORACLE_HOME/rdbms/admin/awrrpt.sql
@$ORACLE_HOME/rdbms/admin/awrsqrpt.sql
```

手动创建快照

```
exec dbms_workload_repository.create_snapshot;
```

删除当前SQLID的执行计划

```
select 'exec sys.dbms_shared_pool.purge('''||address||','||hash_value||''',''C'');' 
from v$sqlarea where sql_id='19sxt3v07nzm4';
```