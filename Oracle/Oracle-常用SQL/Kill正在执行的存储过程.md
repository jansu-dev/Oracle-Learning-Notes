#### Kill 正在执行的PROCEDURE
1. ##### 查询正在执行的存储过程另外一种方法:  

  V$DB_OBJECT_CACHE displays database objects that are cached in the library cache. Objects include tables, indexes, clusters, synonym definitions, PL/SQL procedures and packages, and triggers.
```
select owner,name from v$db_object_cache 
where type like '%PROCEDURE%' and locks >0 and pins >0;
```


2. ##### 通过PROCEDURE定位到sid ,serial# 

  定位
```shell
select b.sid,b.SERIAL#,a.OBJECT,'alter system kill session '
 || '''' || b.sid || ',' ||b.SERIAL# ||  ''';' kill_command
 from   SYS.V_$ACCESS a, SYS.V_$session b
 where    a.type = 'PROCEDURE'
 and   (a.OBJECT like upper('%PROCEDURE名%') 
 or a.OBJECT like lower('%PROCEDURE名%'))
 and a.sid = b.sid
 and b.status = 'ACTIVE';
```

3. ##### 以PROCEDURE的sid ,serial# 杀死回话  
```
alter system kill session 'sid,SERIAL#';
```