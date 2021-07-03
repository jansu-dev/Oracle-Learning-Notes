---
title:[Oracle]--数据迁移之使用数据泵迁移数据库至ADG新库
date:2020-06-15
---



##### 数据泵迁移数据库

### 准备工作

##### [源端节点]--查询所有非数据库默认用户

```
select username
  from dba_users
 where username not in ('MGMT_VIEW',
 'SYS','SYSTEM','DBSNMP','SYSMAN', 'SCOTT','OUTLN',
'FLOWS_FILES','MDSYS', 'ORDSYS','EXFSYS','WMSYS', 'APPQOSSYS',
'APEX_030200','OWBSYS_AUDIT','CTXSYS','ORDDATA', 'ANONYMOUS',
'XDB', 'ORDPLUGINS','OWBSYS','SI_INFORMTN_SCHEMA''OLAPSYS',
'ORACLE_OCM','XS$NULL', 'BI''PM','MDDATA','IX', 'SH''DIP','OE',
'HR','APEX_PUBLIC_USER','SPATIAL_CSW_ADMIN_USR' 'SPATIAL_WFS_ADMIN_USR');
```

##### [源端节点]--查询大表

```
set linesize 120;
select *
  from (select segment_name, bytes / 1024 / 1024 / 1024 tablesize
          from dba_segments
         order by tablesize desc) t
 where rownum <= 10;
```

##### [目标节点]--创建用户

```
create user user1 identified by oracle;
```



### 导出主要大表部分数据

##### [源端节点]--创建用户导出大表数据

```
expdp scott/scott dumpfile=user1_tables.dmp directory=DIR logfile=user1_tables_expdp.log job_name=20200615 tables=t1 query="'where status=''VALID'' '";
```

##### [源端节点]--传输数据到目标端

```
scp /DIR对应OS目录下/user1_tables.dmp oracle@目标端IP:/目标端DIR对应OS目录
```

##### [目标节点]--为使数据库信息一致重建用户,导入数据

```
drop user scott cascade;

create user scott identified by scott account unlock;

grant connect,resource to scott;


impdp \"/ as sysdba\" dumpfile=user1_tables.dmp directory=DIR logfile=user1_tables_impdp.log job_name=20200615;
```



### 基于用户数据迁移

##### [源端节点]--导出用户,

注意：注意：排除的表名一定要大写，否则不识别。

```
expdp \"/ as sysdba\" dumpfile=user1_lack_tables.dmp directory=DIR logfile=user1_lack_expdp.log job_name=20200615 schemas=scott exclude=table:"in('T1')";
```

##### [源端节点]--传输用户数据到目标端

```
scp /DIR对应OS目录下/user1_lack_tables.dmp oracle@目标端IP:/目标端DIR对应OS目录
```

##### [目标节点]--导入用户

注意：表采用append追加模式,默认也是append追加模式

```
impdp \"/ as sysdba\" dumpfile=user1_lack_tables.dmp directory=DIR logfile=user1_lack_impdp.log job_name=20200615; 
```

#### 交割当天追加增量

##### [源端节点]--导出大表增量部分

```
expdp scott/scott dumpfile=user1_last_tables.dmp directory=DIR logfile=user1_tables_last_expdp.log job_name=20200615 tables=t1 content=data_only query="'where status=''INVALID'' '";
```

##### [源端节点]--传输用户数据到目标端

```
scp /DIR对应OS目录下/user1_last_tables.dmp oracle@目标端IP:/目标端DIR对应OS目录
```

##### [目标节点]--追加用户数据

注意：表导入采用append追加模式,默认也是append追加模式

```
impdp \"/ as sysdba\" dumpfile=user1_last_tables.dmp directory=DIR logfile=user1_tables
```