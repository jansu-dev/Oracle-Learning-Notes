---
title: [Oracle]--ORA-002998父键不存在问题解决
date: 2020-07-02
---



ORA-002998″cannot validate (XXX) – parent keys not found”



### 实验触发02298

```
-- 准备数据
conn scott/scott

select * from user_tables;

create table t1(id_t1 int,name varchar2(10));

alter table t1 add primary key (id_t1);

create table t2(id_t2 int,name varchar2(10));

insert into t1 values (0,'0');

insert into t1 values (1,'1');

insert into t2 values (1,'1');

insert into t2 values (2,'2');

commit;

select * from t1;

select * from t2;

-- 创建约束，触发02298
alter table t2 add constraint pk_id_t1 foreign key(id_t2) references t1(id_t1) enable;


select * from user_cons_columns;

select * from user_constraints;

-- 删除外键表多余行
delete from t2 where id_t2 not in (select id_t1 from t1);

commit;


-- 成功创建约束

alter table t2 add constraint pk_id_t1 foreign key(id_t2) references t1(id_t1) enable;

-- 删除约束命令
alter table t2 drop constraint pk_id_t1;
```



### 测试库修复

**生产库慎用**

```sql
--删除外键表中，主键表参照字段不存在的行
SELECT 'delete from '||a.table_name||' a where not exists ( select 1 from '||c_pk.table_name|| 'b where b.'|| b.column_name||'=a.'||a.column_name||');'
FROM user_cons_columns a
JOIN user_constraints c
ON a.constraint_name = c.constraint_name
JOIN user_constraints c_pk
ON c.r_constraint_name = c_pk.constraint_name
JOIN user_cons_columns b
ON c_pk.constraint_name = b.constraint_name
WHERE c.constraint_type = 'R'
AND a.table_name = 'T1'
AND a.constraint_name ='ID_T2';


--执行enable外键的操作
select ‘alter table ‘||owner||’.’||table_name||’ enable constraint ‘||constraint_name||’;’
from dba_constraints
where owner=’XXX’ and constraint_type=’R’ and status=’DISABLED’;
```