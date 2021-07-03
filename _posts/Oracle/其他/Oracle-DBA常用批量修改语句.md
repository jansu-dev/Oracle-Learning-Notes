---
title:Oracle-DBA常用批量修改语句
date:2020-08-19
---



Oracle-DBA常用批量修改语句



### 批量操作

批量重建某表索引

```
select 'alter index '||owner||'.'||index_name||' rebuild nologging parallel 8;' from dba_indexes where table_name='UNUSABLE';
```

批量删索引

```
select 'drop index '||owner||'.'||index_name||';' from dba_indexes where table_name in ('ASSIGN_POLICY','ASSIGN_POLICY_ITEM','TBL_UNIT_FIELD','TBL_USER_POLICY_RELATION','TBL_ALL_BILL_USER_POLICY','TBL_CATALOGUE_POLICY','TBL_POLICY_ITEM','TBL_SYS_DEPARTMENTS') and owner = 'EMSHEET';
```

批量删约束

```
select 'alter table '||owner||'.'|| table_name || ' drop constraint ' || constraint_name||';' from dba_constraints where table_name in ('ASSIGN_POLICY','ASSIGN_POLICY_ITEM','TBL_UNIT_FIELD','TBL_USER_POLICY_RELATION','TBL_ALL_BILL_USER_POLICY','TBL_CATALOGUE_POLICY','TBL_POLICY_ITEM','TBL_SYS_DEPARTMENTS') and owner = 'EMSHEET';
```

批量改名字

```
select 'alter table '||owner||'.'||table_name||' drop constrain
```