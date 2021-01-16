---
title: Oracle查询表空间使用情况以及其他查询
date: 2019-10-13
categories: Oracle
tags: Oracle常见问题
---



#


1. ##### 查询表空间的free space

  select tablespace_name,
  　　count(*) as extends,
  　　round(sum(bytes) / 1024 / 1024, 2) as MB,
  　　sum(blocks) as blocks
  　　from dba_free_space
  　　group by tablespace_name;

​    


2. ##### 查询表空间的总容量

  select tablespace_name, sum(bytes) / 1024 / 1024 as MB
  　　from dba_data_files
  　　group by tablespace_name;

​    

3. ##### 查询表空间使用率

  select total.tablespace_name,
  		round(total.MB, 2) as Total_MB,考试大论坛
  　　round(total.MB - free.MB, 2) as Used_MB,
  　　round((1 - free.MB / total.MB) * 100, 2) || '%' as Used_Pct
  　　from (select tablespace_name, sum(bytes) / 1024 / 1024 as MB
  　　from dba_free_space
  　　group by tablespace_name) free,
  　　(select tablespace_name, sum(bytes) / 1024 / 1024 as MB
  　　from dba_data_files
  　　group by tablespace_name) total
  　　where free.tablespace_name = total.tablespace_name;

​    

4. ##### 查找当前表级锁的SQL如下

   select sess.sid, 
   sess.serial#, 
   lo.oracle_username, 
   lo.os_user_name, 
   ao.object_name, 
   lo.locked_mode 
   from v$locked_object lo, 
   dba_objects ao, 
   v$session sess 
   where ao.object_id = lo.object_id and lo.session_id = sess.sid;  

  

5. ##### 杀掉锁表进程

  alter system kill session '436,35123';   

  

6. ##### RAC环境中锁查找

  SELECT inst_id,DECODE(request,0,'Holder: ','Waiter: ')||sid sess, 
  	id1, id2, lmode, request, type,block,ctime
  FROM GV$LOCK
  WHERE (id1, id2, type) IN
  	(SELECT id1, id2, type FROM GV$LOCK WHERE request>0)
  ORDER BY id1, request;  

  

7. ##### 监控当前数据库谁在运行什么SQL语句 

  select osuser, username, sql_text  
  from  v$session a, v$sqltext b 
  where  a.sql_address =b.address order by address, piece;  

  

8. ##### 找使用CPU多的用户session 

  select a.sid,spid,status,substr(a.program,1,40) prog, a.terminal,osuser,value/60/100 value 
  from  v$session a,v$process b,v$sesstat c 
  where  c.statistic#=12 and  
      c.sid=a.sid and  
      a.paddr=b.addr  
      order by value desc;  

  

9. ##### 查看死锁信息

  SELECT (SELECT username
         FROM v$session
        WHERE SID = a.SID) blocker, a.SID, 'is blocking',
      (SELECT username
         FROM v$session
        WHERE SID = b.SID) blockee, b.SID
    FROM v$lock a, v$lock b
   WHERE a.BLOCK = 1 AND b.request > 0 AND a.id1 = b.id1 AND a.id2 = b.id2;  

  

10. ##### 具有最高等待的对象

    SELECT   o.OWNER,o.object_name, o.object_type, a.event,
    	SUM (a.wait_time + a.time_waited) total_wait_time
    FROM v$active_session_history a, dba_objects o
    	WHERE a.sample_time BETWEEN SYSDATE - 30 / 2880 AND SYSDATE
    AND a.current_obj# = o.object_id
    GROUP BY o.OWNER,o.object_name, o.object_type, a.event
    ORDER BY total_wait_time DESC;
    SELECT   a.session_id, s.osuser, s.machine, s.program, o.owner, o.object_name,
    	o.object_type, a.event,
        SUM (a.wait_time + a.time_waited) total_wait_time
    FROM v$active_session_history a, dba_objects o, v$session s
    	WHERE a.sample_time BETWEEN SYSDATE - 30 / 2880 AND SYSDATE
    AND a.current_obj# = o.object_id
    AND a.session_id = s.SID
    GROUP BY o.owner,
    	o.object_name,
    	o.object_type,
    	a.event,
    	a.session_id,
    	s.program,
    	s.machine,
    	s.osuser
    ORDER BY total_wait_time DESC;  

    

11. ##### 查询当前连接会话数

    select s.value,s.sid,a.username
    from 
    v$sesstat S,v$statname N,v$session A
    where 
    n.statistic#=s.statistic# and
    name='session pga memory'
    and s.sid=a.sid
    order by s.value;  

    

12. ##### 等待最多的用户

    SELECT   s.SID, s.username, SUM (a.wait_time + a.time_waited) total_wait_time
    FROM v$active_session_history a, v$session s
       WHERE a.sample_time BETWEEN SYSDATE - 30 / 2880 AND SYSDATE
    GROUP BY s.SID, s.username
    ORDER BY total_wait_time DESC;  

    

13. ##### 等待最多的SQL

    SELECT   a.program, a.session_id, a.user_id, d.username, s.sql_text,
    	SUM (a.wait_time + a.time_waited) total_wait_time
    FROM v$active_session_history a, v$sqlarea s, dba_users d
    	WHERE a.sample_time BETWEEN SYSDATE - 30 / 2880 AND SYSDATE
     AND a.sql_id = s.sql_id
     AND a.user_id = d.user_id
    GROUP BY a.program, a.session_id, a.user_id, s.sql_text, d.username;

    

14. ##### 查看消耗资源最多的SQL

    SELECT hash_value, executions, buffer_gets, disk_reads, parse_calls
    FROM V$SQLAREA
    WHERE buffer_gets > 10000000 OR disk_reads > 1000000
    ORDER BY buffer_gets + 100 * disk_reads DESC;

    

15. ##### 查看某条SQL语句的资源消耗

    SELECT hash_value, buffer_gets, disk_reads, executions, parse_calls
    FROM V$SQLAREA
    WHERE hash_Value = 228801498 AND address = hextoraw('CBD8E4B0');、

    

16. ##### 查询会话执行的实际SQL

    SELECT   a.SID, a.username, s.sql_text
    FROM v$session a, v$sqltext s
    	WHERE a.sql_address = s.address
     AND a.sql_hash_value = s.hash_value
     AND a.status = 'ACTIVE'
    ORDER BY a.username, a.SID, s.piece;

    

17. ##### 显示正在等待锁的所有会话

    SELECT * FROM DBA_WAITERS;

