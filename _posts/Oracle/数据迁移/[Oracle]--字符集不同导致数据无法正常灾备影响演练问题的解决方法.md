---
title: [Oracle]--字符集不同导致数据无法正常灾备影响演练问题的解决方法
date: 2020-09-03
---

​		

​		客户系统要进行灾备演练，需要将某省使用的数据切换至备用数据库，并启用原备份数据库为生产库，但却出现数据无法导入的情况。

### 问题原因

因为源端采用字符集是ZHS16gbk，目标端使用的字符集是AL32UTF8；
一个汉字在AL32UTF8中占4个字节，在ZHS16gbk中占三个字节
所以在导入的时候存在汉字宽度综合大于字段宽度限制无法导入的情况；

### 解决步骤

本次采用的解决办法是依据JAN_ZZZZ4_XXX_20200828_02.log日志中的错误信息，
相对应的修改问题表的字段宽度后重新导入的方式解决。

1. 目标端使用数据泵导出备份
   expdp "/ as sysdba" directory=EXPDP_DIR dumpfile=ZZZZ4_XXX_wuhan_20200828_backup.dmp
   job_name=20200828 logfile=ZZZZ4_XXX_wuhan_20200828_backup.log schemas=ZZZZ4_XXX
2. 源端端导出备份：

```
expdp \"/ as sysdba\" directory=DUMP_DIR dumpfile=jan_ZZZZ4_XXX_20200828_bak.dmp job_name=20200828 
logfile=jan_ZZZZ4_XXX_20200828.log schemas=ZZZZ4_XXX
```

​	3. 目标端导入源数据：

```
impdp \"/ as sysdba\" directory=EXPDP_DIR dumpfile=jan_ZZZZ4_XXX_20200828.dmp job_name=20200828 
logfile=jan_ZZZZ4_XXX_20200828_01.log schemas=ZZZZ4_XXX remap_tablespace=YY_XXX:YY_CC content=metadata_only
```

​	 导入数据之后，发现视图不可用。

```
select count(*) from XXX_KB_DAY_HITS;
select count(*) from XXX_KB_ALL_HITS;
select count(*) from XXX_KB_TAG_COUNTS;
select count(*) from XXX_KB_TAG_HITS;
```

​	4. <span style="color:red">**依据jan_ZZZZ4_XXX_20200828_02.log日志中的错误信息，修改有问题的表字段宽度定义！！！**</span>

​	5. 拼接truncate语句将物化视图以外数据删除：

```
select 'truncate table '||owner||'.'||table_name||';' from dba_tables where owner='ZZZZ4_XXX'
and table_name not in ('XXX_KB_DAY_HITS','XXX_KB_ALL_HITS','XXX_KB_TAG_COUNTS','XXX_KB_TAG_HITS')
```

​	6. 目标端重新导入数据：

```
impdp \"/ as sysdba\" directory=EXPDP_DIR dumpfile=jan_ZZZZ4_XXX_20200828.dmp job_name=20200828 
logfile=jan_ZZZZ4_XXX_20200828_02.log schemas=ZZZZ4_XXX remap_tablespace=YY_XXX:YY_CC content=data_only
```

	7. 重建物化视图
	    在同事的的帮助下，找到当初的物化视图初始化语句重新初始化。

```
prompt
prompt Creating materialized view XXX_kb_day_hits
prompt ============================================
prompt
create materialized view log on XXX_kb_traffic with rowid ,sequence  (domain_id, traffic_id, user_id, object_id, object_type, action, hit_time, source) including new values
/
create MATERIALIZED VIEW XXX_kb_day_hits
 build deferred REFRESH fast start with sysdate+1/24/60 next sysdate+1/24 as
 select
  t.domain_id,
  t.user_id,
        t.object_id,
        t.object_type,
        trunc(t.hit_time) as hit_time,
        count(*) as hits,
  t.action
   from XXX_kb_traffic t
   where t.action in ('XXX.kb.action.read', 'XXX.kb.action.download')
  group by
  t.domain_id,
        t.user_id,
        t.object_id,
        t.object_type,
        trunc(t.hit_time),
  t.action
/
prompt
prompt Creating materialized view XXX_kb_all_hits
prompt ============================================
prompt
create MATERIALIZED VIEW XXX_kb_all_hits
 build deferred REFRESH fast start with sysdate+1/24/60 next sysdate+1/24 as
 select
  t.domain_id,
        t.object_id,
        t.object_type,
        count(*) as hits,
  t.action
   from XXX_kb_traffic t
   where t.action in ('XXX.kb.action.read', 'XXX.kb.action.download')
  group by
  t.domain_id,
        t.object_id,
        t.object_type,
  t.action
/
exec dbms_mview.refresh('XXX_kb_all_hits','c')
/
prompt
prompt Creating materialized view XXX_kb_tag_counts
prompt ============================================
prompt
create materialized view log on XXX_kb_tag with rowid ,sequence  (domain_id, tag_id, object_id, object_type, object_state, tag) including new values
/
create MATERIALIZED VIEW XXX_kb_tag_counts
 build deferred REFRESH fast start with sysdate+1/24/60 next sysdate+1/24 as
select
  t.domain_id,
  t.tag,
        count(*) as counts
   from XXX_kb_tag t
   where t.object_state = 'release' and t.tag is not null
  group by t.domain_id, t.tag;
/
exec dbms_mview.refresh('XXX_kb_tag_counts','c')
/
prompt
prompt Creating materialized view XXX_kb_tag_hits
prompt ============================================
prompt
create materialized view log on XXX_kb_tag_traffic with rowid ,sequence  (domain_id, traffic_id, user_id, tag, hit_time, source) including new values
/
create MATERIALIZED VIEW XXX_kb_tag_hits
 build deferred REFRESH fast start with sysdate+1/24/60 next sysdate+1/24 as
 select
  t.domain_id,
  t.tag,
        count(*) as hits
   from XXX_kb_tag_traffic t
  group by t.domain_id, t.tag
/
exec dbms_mview.refresh('XXX_kb_tag_hits','c')
/
```



### 案例总结

主备库最好采用相同字符集，否则很容易踩坑；
DBA应该及时检查备份情况，早发现早处理。