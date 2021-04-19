---
title:[Oracle]-带有集合函数的视图会阻止谓词下推
date:2020-12-18
---

### 性能问题

​    客户反馈页面查不出数据，大致定位原因时SQL跑不出来，于是反馈到DBA数据库层面。

![image.png](http://cdn.lifemini.cn/dbblog/20201222/1dbad336923441f496db58f07a674995.png)

![image.png](http://cdn.lifemini.cn/dbblog/20201222/306616b0d19145aeaf1ab078a472979a.png)

​    通过AWR及相关用户定位到SQL如下,结合业务发现两个主要的SQL的功能是分页和生成页面数据；

​    SQL语句如下：

```
SELECT
        distinct contactid,
        s.oriani,
        s.oridnis,
        s.agentid,
        s.agent_name,
        s.user_id,
        s.skillid,
        s.skillname,
        s.region,
        s.reportskillname,
        s.dept_code,
        s.dept_name,
        s.starttime starttime,
        s.endtime endtime,
        s.day_id,
        sum(s.repeatcalls)keep(dense_rank last order by exec_dayid)over(partition by s.contactid)repeatcalls
    FROM
        XXX_yreport.t_report_once_solve_rate24 s  -- where s.oriani = '136******98'
    ORDER BY
        s.oriani,
        s.starttime
```

​    执行计划如下：
![image.png](http://cdn.lifemini.cn/dbblog/20201222/ec630dfe6bcb4eb79d904dc87789133b.png)

​    可见XXX_report.t_report_once_solve_rate24是基于V_REPORT_ONCE_SOLVE_RATE24_DAY基表建立的视图；但是在基表T_REPORT_ONCE_SOLVE_RATE24的DAY_ID列上是存在索引的全表数据在1亿条所有，过滤后的结果集在1万条左右，原理上应该会选择索引获取小的结果集，但是却没有！！！

​    视图定义里面如下：

```
CREATE OR REPLACE VIEW XXX_REPORT.V_REPORT_ONCE_SOLVE_RATE24_DAY AS
SELECT
        distinct contactid,
        s.oriani,
        s.oridnis,
        s.agentid,
        s.agent_name,
        s.user_id,
        s.skillid,
        s.skillname,
        s.region,
        s.reportskillname,
        s.dept_code,
        s.dept_name,
        s.starttime starttime,
        s.endtime endtime,
        s.day_id,
        sum(s.repeatcalls)keep(dense_rank last order by exec_dayid)over(partition by s.contactid)repeatcalls
    FROM
        t_report_once_solve_rate24 s  -- where s.oriani = '136******98'
    ORDER BY
        s.oriani,
        s.starttime
;
```

​    视图定义的select部分的执行计划如下：

```
--------------------------------------------------------------------------------------------------------
| Id | Operation             | Name                       | Rows     | Bytes      | Cost    | Time     |
--------------------------------------------------------------------------------------------------------
|  0 | SELECT STATEMENT      |                            | 10830155 | 2415124565 | 1680438 | 05:36:06 |
|  1 |   SORT UNIQUE         |                            | 10830155 | 2415124565 | 1156600 | 03:51:20 |
|  2 |    WINDOW SORT        |                            | 10830155 | 2415124565 | 1680438 | 05:36:06 |
|  3 |     TABLE ACCESS FULL | T_REPORT_ONCE_SOLVE_RATE24 | 10830155 | 2415124565 |  108925 | 00:21:48 |
--------------------------------------------------------------------------------------------------------
```

### 猜想假设

​    猜想视图的定义包含sum(s.repeatcalls)keep(dense_rank last order by exec_dayid)over(partition by s.contactid)repeatcalls部分，正是这部分导致Oracle优化器认为将谓词下推后的结果与下推之前不等价，故而未能将谓词下推。

### 生产验证

​    将原有视图部分定义直接嵌套至生产的慢SQL之中，将视图的sum部分注释掉；

​    修改后SQL如下：

```
select count(1) as cnt
  from (SELECT s.oriani      AS FIELD_1,
               s.agentid     AS FIELD_2,
               s.agent_name  AS FIELD_3,
               s.starttime   AS FIELD_4,
               s.endtime     AS FIELD_5/*,
               s.repeatcalls AS FIELD_6*/
          FROM (SELECT distinct contactid,
                                s.oriani,
                                s.oridnis,
                                s.agentid,
                                s.agent_name,
                                s.user_id,
                                s.skillid,
                                s.skillname,
                                s.region,
                                s.reportskillname,
                                s.dept_code,
                                s.dept_name,
                                s.starttime starttime,
                                s.endtime endtime,
                                s.day_id/*,
                                sum(s.repeatcalls) keep(dense_rank last order by exec_dayid) over(partition by s.contactid) repeatcalls*/
                  FROM XXX_report.t_report_once_solve_rate24 s -- where s.oriani = '13641399198'
                /* ORDER BY s.oriani, s.starttime*/) s
         WHERE s.day_id < to_char(sysdate - 1, 'yyyy-mm-dd')
           AND S.DAY_ID <= '2020-12-18'
           AND S.DAY_ID >= '2020-12-14') t_1
```

​    谓词下退后的执行计划；
![image.png](http://cdn.lifemini.cn/dbblog/20201222/3eab1b885a724568a659944d302b651d.png)



### 归纳总结

​    可见，Oracle优化器在使用视图且视图定义中包含聚合函数时，优化器认为二者不等价，故而不能很好下推。