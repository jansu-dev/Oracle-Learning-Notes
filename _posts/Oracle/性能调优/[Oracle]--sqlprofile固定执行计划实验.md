---
title:[Oracle]--sqlprofile固定执行计划实验
date:2020-07-21
---

​		

​		为确保SQL语句保持正确的执行计划，在某些情况下，可以使用sql profile在数据库层面固定SQL语句的执行计划。

- [创建测试表](创建测试表)
- [查看执行计划](查看执行计划)
- [获取源计划大纲](获取源计划大纲)
- [改变执行计划](改变执行计划)
- [绑定执行计划](绑定执行计划)
- [删除命令](删除命令)



### 创建测试表

根据DBA_OBJECTS创建测试表，并在OBJECT_ID上建立索引。

```
Create table jan_tbd as select * from dba_objects;

create index t_3 on jan_tbd(object_id);
```



### 查看执行计划

查看SQL默认执行计划,走了索引

```
explain plan for select * from jan_tbd where object_id= :a;
```



### 获取源计划大纲

通过指定outline可以获取到系统为我们生成的hint。

```
select * from table(dbms_xplan.display(null,null,'outline'));


-----------------------------------------------

| Id  | Operation                   | Name    |

-----------------------------------------------

|   0 | SELECT STATEMENT            |         |

|   1 |  TABLE ACCESS BY INDEX ROWID| JAN_TBD |

|*  2 |   INDEX RANGE SCAN          | T_3     |

-----------------------------------------------

Outline Data

-------------



  /*+

      BEGIN_OUTLINE_DATA

      INDEX_RS_ASC(@"SEL$1" "JAN_TBD"@"SEL$1" ("JAN_TBD"."OBJECT_ID"))

      OUTLINE_LEAF(@"SEL$1")

      ALL_ROWS

      DB_VERSION('11.1.0.7')

      OPTIMIZER_FEATURES_ENABLE('11.1.0.7')

      IGNORE_OPTIM_EMBEDDED_HINTS

      END_OUTLINE_DATA

  */
```



### 改变执行计划

如果我们想让它走全表扫描，获取全表扫描HINT

```
explain plan for select /*+ full(JAN_tbd) */* from JAN_tbd where object_id= :a;-----------增加HINT

select * from table(dbms_xplan.display(null,null,'outline'));------------可以看到全表扫描的hint已经为我们生成了，我们选取必要的hint就OK了，其他的可以不要

-------------------------------------

| Id  | Operation         | Name    |

-------------------------------------

|   0 | SELECT STATEMENT  |         |

|*  1 |  TABLE ACCESS FULL| JAN_TBD |

-------------------------------------

Outline Data

-------------



  /*+

      BEGIN_OUTLINE_DATA

     FULL(@"SEL$1" "JAN_TBD"@"SEL$1")

      OUTLINE_LEAF(@"SEL$1")

      ALL_ROWS

      DB_VERSION('11.1.0.7')

      OPTIMIZER_FEATURES_ENABLE('11.1.0.7')

      IGNORE_OPTIM_EMBEDDED_HINTS

      END_OUTLINE_DATA

  */
```



### 绑定执行计划

查看是否生效，已经生效了。

```
使用sql profile

declare

  v_hints sys.sqlprof_attr;

begin

  v_hints := sys.sqlprof_attr('FULL(@"SEL$1" "JAN_TBD"@"SEL$1")');----------从上面Outline Data部分获取到的HINT

  dbms_sqltune.import_sql_profile('select * from JAN_tbd where object_id= :a',----------SQL语句部分

                                  v_hints,

                                 'JAN_TBD',--------------------------------PROFILE的名字

                                  force_match =>true);

end;

/
```



### 删除命令

```
begin
 dbms_sqltune.drop_sql_profile(name =>'SQLPROFILE_LEE1');
end;
```