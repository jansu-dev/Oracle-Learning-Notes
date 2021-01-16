---
title: 应用发版新SQL缺索引导致全表扫
date: 2020-06-05 03:02:38
tags:
---



性能调优很大比重在SQL调优，提到SQL调优一定离不开的就是所以优化。
索引优化可以说是DBA在数据库最常见的优化操作。

* 数据库环境介绍
* 定位慢SQL
* 索引优化原理
* 优化SQL

得知数据库卡顿后，采集了8:00~9:00这段时间的ASH报告。

![收缩日志空间](http://cdn.lifemini.cn/dbblog/20210115/15e64dd8b54b40a3892b871062cc96f2.png)





### 1.数据库环境介绍

数据库版本：11.2.0.3.0
RAC: 是




### 2.定位慢SQL
从报告上可以得知红框部分SQL有问题<span style="color:red">“TABLE ACCESS FULL”</span>。虽然前面有执行次数更多的慢SQL，<span style="color:green">排查结果是其他库的dblink取数导致前面慢SQL,且谓词部分难以优化,所以暂不处理</span>，本篇文章着重圈出部分SQL区域的问题。


![收缩日志空间](http://cdn.lifemini.cn/dbblog/20210115/0cbd9f9b643742d5b659ef3aeb6a8217.png)


根据SQL_ID使用DBMS_XPLAN.DISPLAY_AWR包取出当时此SQL的历史执行计划，


```sql
select * from table(dbms_xplan.display_awr('22pcpwpapvz2f'));




SQL_ID 22pcpwpapvz2f
--------------------
 
Plan hash value: 2679156405
 
--------------------------------------------------------------------------------------------------
| Id  | Operation             | Name                     | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |                          |       |       |  5059K(100)|          |
|   1 |  SORT ORDER BY        |                          |     4 |   192 |  5059K  (1)| 16:51:57 |
|   2 |   NESTED LOOPS        |                          |     4 |   192 |  5059K  (1)| 16:51:57 |
|   3 |    VIEW               |                          |     1 |    13 |  2529K  (1)| 08:25:59 |
|   4 |     SORT AGGREGATE    |                          |     1 |    50 |            |          |
|   5 |      TABLE ACCESS FULL| XXXX_TICKET_DETAIL       |  4465 |   218K|  2529K  (1)| 08:25:59 |
|   6 |    VIEW               |                          |     4 |   140 |  2529K  (1)| 08:25:59 |
|   7 |     SORT GROUP BY     |                          |     4 |   260 |  2529K  (1)| 08:25:59 |
|   8 |      TABLE ACCESS FULL| XXXX_TICKET_DETAIL       |   454 | 29510 |  2529K  (1)| 08:25:59 |
--------------------------------------------------------------------------------------------------
```
从执行计划可以看出,SQL瓶颈在XXXX_TICKET_DETAIL表的两个子查询的执行计划均走全表扫，消耗大量I/O，导致SQL执行过慢。下图红框圈出部分就是对应的子查询SQL语句。

定位问题后，对于全表扫通常使用的手段是建索引解决此类问题。

![收缩日志空间](http://cdn.lifemini.cn/dbblog/20210115/54063ba723c8428daa9e67cae5ce15f8.png)

使用GROUP BY查询不同谓词的选择性，具体查询结果便不在此贴出。

```sql
--使用如下SQL查改该表不同字段的选择性
select /*+parallel(6)*/count(*),
       ticket_type,
       province_code,
       city_code,
       county_code,
       biz_type,
       mail_kind_code,
       ep_code_id
  from XXXX_ticket_detail
 group by ticket_type,
          province_code,
          city_code,
          county_code,
          biz_type,
          mail_kind_code,
          ep_code_id;



--使用如下SQL查询XXXX_ticket_detail总数据量大概为4亿左右
select /*+parallel(6)*/count(*) from XXXX_ticket_detail;
```



### 3.索引优化原理

最后确定这三个(province_code,city_code,county_code)表示地区的字段以及create_time适合建立索引。
因为子查询的最后一个过滤条件为created_time lik '2020-05%'，将created_time作为索引的最后一个字段时，会直接在索引中排序。

最后，确定采用county_code和create_time两个字段作为索引条件。

<div style="color:red">
注意：
</br>
1.country_code代表区号，每个区都有不同的区号，索引选择性更好。
</br>
2.create_time紧随country_code之后建立索引，执行这条SQL语句扫描索引时，会直接遍历B+Tree结构索引的所有叶子节点，从而降低rowid数量\减少回表、降低I/O。
</br>
</div>



### 4.优化SQL

```
--使用并行创建组合索引
create index XXX_USER.ind_city_combine on XXXX_ticket_detail(city_code,create_time) nologging parallel 6;

--关闭索引并行
alter index XXX_USER.ind_city_combine noparallel;



 Plan Hash Value  : 2056268969 

---------------------------------------------------------------------------------------------------------
| Id   | Operation                          | Name                     | Rows | Bytes | Cost | Time     |
---------------------------------------------------------------------------------------------------------
|    0 | SELECT STATEMENT                   |                          |    1 |    48 | 3463 | 00:00:42 |
|    1 |   SORT ORDER BY                    |                          |    1 |    48 | 3463 | 00:00:42 |
|    2 |    MERGE JOIN CARTESIAN            |                          |    1 |    48 | 3462 | 00:00:42 |
|    3 |     VIEW                           |                          |    1 |    35 | 1732 | 00:00:21 |
|    4 |      HASH GROUP BY                 |                          |    1 |    65 | 1732 | 00:00:21 |
|  * 5 |       TABLE ACCESS BY INDEX ROWID  | XXXX_TICKET_DETAIL       |    1 |    65 | 1731 | 00:00:21 |
|  * 6 |        INDEX RANGE SCAN            | IND_COUNTRY_COMBINE      | 7838 |       |   10 | 00:00:01 |
|    7 |     BUFFER SORT                    |                          |    1 |    13 | 3463 | 00:00:42 |
|    8 |      VIEW                          |                          |    1 |    13 | 1731 | 00:00:21 |
|    9 |       SORT AGGREGATE               |                          |    1 |    50 |      |          |
| * 10 |        TABLE ACCESS BY INDEX ROWID | XXXX_TICKET_DETAIL       |    1 |    50 | 1731 | 00:00:21 |
| * 11 |         INDEX RANGE SCAN           | IND_COUNTRY_COMBINE      | 7838 |       |   10 | 00:00:01 |
---------------------------------------------------------------------------------------------------------
```
<div style="color:green">
建立索引后的SQL语句执行计划给出的参考时间为42秒，这是由于SQL执行计划以及相应的数据没有缓存入SGA导致的，实际执行SQL时间为0.136秒。
</div>

