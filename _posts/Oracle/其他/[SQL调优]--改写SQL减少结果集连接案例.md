---
title:[SQL调优]--改写SQL减少结果集连接案例
date:2020-06-23
---



#### 目录

- ##### [改写之前](改写之前)

- ##### [改写之后](改写之后)

- ##### [效果对比](效果对比)

- ##### [改写原理](改写原理)







##### SQL调优回行改写

### 改写之前

**企业信息以进行“XXXX_”处理**

```
select V.ID,
       ... 
       ...
  FROM (select t.ID,
               ... 
               ... 
               t.levelcode,
               row_number() over(partition by reconn_first_id order by create_time desc) num
          from XXXX_visitor t) v
  LEFT JOIN (select reconn_first_id,
                    client_id,
                    max(agent_id) agent_id,
                    max(answer_time) answer_time,
                    sum(agent_msg_count) as agent_msg_count,
                    sum(visitor_msg_count) as visitor_msg_count
               from XXXX_answer
              where CREATE_TIME >= '2020-06-03 00:00:00'
                and CREATE_TIME <= '2020-06-03 23:59:59'
              group by reconn_first_id, client_id) a
    ON v.client_id = a.client_id
  LEFT JOIN (SELECT D.RECONN_FIRST_ID AS CLIENT_ID,
                    MAX(D.DISCONNECT_TIME) AS DISCONNECT_TIME,
                    MAX(D.PUSH_MSG) AS PUSH_MSG
               FROM XXXX_VISITOR_DISCONNECT D
              GROUP BY D.RECONN_FIRST_ID) D
    ON V.CLIENT_ID = D.CLIENT_ID
  LEFT JOIN XXXX_biz b
    ON v.client_id = b.client_id
  LEFT JOIN XXXX_WX_CUSTOMER C
    ON V.OPEN_ID = C.OPEN_ID
  LEFT JOIN XXXX_SATISFACTION S
    ON S.CUSTOMER_SESSION_ID = B.CLIENT_ID
 where 1 = 1
   and (V.CREATE_TIME <= '2020-06-03 23:59:59' and
       V.CREATE_TIME >= '2020-06-03 00:00:00')
   and v.reconn = '0'
   and b.skill is not null
   AND v.user_type = 'user'
   and a.visitor_msg_count > 0
   and a.agent_msg_count > 0
 ORDER BY V.CREATE_TIME DESC



Plan Hash Value  : 2689863409 

--------------------------------------------------------------------------------------------------------------
| Id   | Operation                              | Name                      | Rows | Bytes | Cost | Time     |
--------------------------------------------------------------------------------------------------------------
|    0 | SELECT STATEMENT                       |                           |    1 |   459 |   25 | 00:00:01 |
|    1 |   SORT ORDER BY                        |                           |    1 |   459 |   25 | 00:00:01 |
|    2 |    NESTED LOOPS OUTER                  |                           |    1 |   459 |   24 | 00:00:01 |
|    3 |     NESTED LOOPS OUTER                 |                           |    1 |   433 |   19 | 00:00:01 |
|    4 |      NESTED LOOPS                      |                           |    1 |   393 |   15 | 00:00:01 |
|    5 |       NESTED LOOPS OUTER               |                           |    1 |   348 |   12 | 00:00:01 |
|    6 |        NESTED LOOPS                    |                           |    1 |   306 |    8 | 00:00:01 |
|    7 |         VIEW                           |                           |    1 |   129 |    5 | 00:00:01 |
|  * 8 |          FILTER                        |                           |      |       |      |          |
|    9 |           HASH GROUP BY                |                           |    1 |   133 |    5 | 00:00:01 |
|   10 |            TABLE ACCESS BY INDEX ROWID | XXXX_ANSWER               |    1 |   133 |    4 | 00:00:01 |
| * 11 |             INDEX RANGE SCAN           | IND_ANSWER_CTIME          |    1 |       |    3 | 00:00:01 |
| * 12 |         TABLE ACCESS BY INDEX ROWID    | XXXX_VISITOR              |    1 |   177 |    3 | 00:00:01 |
| * 13 |          INDEX RANGE SCAN              | IND_VISITOR_TIME          |    1 |       |    2 | 00:00:01 |
|   14 |        TABLE ACCESS BY INDEX ROWID     | XXXX_WX_CUSTOMER          |    1 |    42 |    4 | 00:00:01 |
| * 15 |         INDEX RANGE SCAN               | INDEX_CUS_OPENID          |    1 |       |    2 | 00:00:01 |
| * 16 |       TABLE ACCESS BY INDEX ROWID      | XXXX_BIZ                  |    1 |    45 |    3 | 00:00:01 |
| * 17 |        INDEX RANGE SCAN                | IND_BIZ_CLI               |    1 |       |    2 | 00:00:01 |
|   18 |      TABLE ACCESS BY INDEX ROWID       | XXXX_SATISFACTION         |    1 |    40 |    4 | 00:00:01 |
| * 19 |       INDEX RANGE SCAN                 | WS_CUSTOMER_SESSION_ID    |    1 |       |    2 | 00:00:01 |
|   20 |     VIEW PUSHED PREDICATE              |                           |    1 |    26 |    5 | 00:00:01 |
|   21 |      SORT GROUP BY                     |                           |    1 |    59 |    5 | 00:00:01 |
|   22 |       TABLE ACCESS BY INDEX ROWID      | XXXX_VISITOR_DISCONNECT   |    1 |    59 |    5 | 00:00:01 |
| * 23 |        INDEX RANGE SCAN                | IND_WECHATDIS_RECONN      |    1 |       |    3 | 00:00:01 |
--------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 8 - filter(SUM(TO_NUMBER("VISITOR_MSG_COUNT"))>0 AND SUM(TO_NUMBER("AGENT_MSG_COUNT"))>0)
* 11 - access("CREATE_TIME">='2020-06-03 00:00:00' AND "CREATE_TIME"<='2020-06-03 23:59:59')
* 12 - filter("T"."RECONN"='0' AND "T"."USER_TYPE"='user' AND "T"."CLIENT_ID"="A"."CLIENT_ID")
* 13 - access("T"."CREATE_TIME">='2020-06-03 00:00:00' AND "T"."CREATE_TIME"<='2020-06-03 23:59:59')
* 15 - access("T"."OPEN_ID"="C"."OPEN_ID"(+))
* 16 - filter("B"."SKILL" IS NOT NULL)
* 17 - access("T"."CLIENT_ID"="B"."CLIENT_ID")
* 19 - access("S"."CUSTOMER_SESSION_ID"(+)="B"."CLIENT_ID")
* 23 - access("D"."RECONN_FIRST_ID"="T"."CLIENT_ID")
```



### 改写之后

```
select V.ID,
       ... 
       ...
  FROM (select t.ID,
               ... 
               ... 
               t.levelcode,
               row_number() over(partition by reconn_first_id order by create_time desc) num
          from XXXX_visitor t) v
  LEFT JOIN (select reconn_first_id,
                    client_id,
                    max(agent_id) agent_id,
                    max(answer_time) answer_time,
                    sum(agent_msg_count) as agent_msg_count,
                    sum(visitor_msg_count) as visitor_msg_count
               from XXXX_answer
              where CREATE_TIME >= '2020-06-03 00:00:00'
                and CREATE_TIME <= '2020-06-03 23:59:59'
                and visitor_msg_count > 0
                and agent_msg_count > 0
              group by reconn_first_id, client_id) a
    ON v.client_id = a.client_id
  LEFT JOIN (SELECT D.RECONN_FIRST_ID AS CLIENT_ID,
                    MAX(D.DISCONNECT_TIME) AS DISCONNECT_TIME,
                    MAX(D.PUSH_MSG) AS PUSH_MSG
               FROM XXXX_VISITOR_DISCONNECT D
              GROUP BY D.RECONN_FIRST_ID) D
    ON V.CLIENT_ID = D.CLIENT_ID
  LEFT JOIN XXXX_biz b
    ON v.client_id = b.client_id
  LEFT JOIN XXXX_WX_CUSTOMER C
    ON V.OPEN_ID = C.OPEN_ID
  LEFT JOIN XXXX_SATISFACTION S
    ON S.CUSTOMER_SESSION_ID = B.CLIENT_ID
 where 1 = 1
   and (V.CREATE_TIME <= '2020-06-03 23:59:59' and
       V.CREATE_TIME >= '2020-06-03 00:00:00')
   and v.reconn = '0'
   and b.skill is not null
   AND v.user_type = 'user'
/* and a.visitor_msg_count > 0
and a.agent_msg_count > 0*/
 ORDER BY V.CREATE_TIME DESC


 Plan Hash Value  : 4130251448 

-----------------------------------------------------------------------------------------------------------
| Id   | Operation                           | Name                      | Rows | Bytes | Cost | Time     |
-----------------------------------------------------------------------------------------------------------
|    0 | SELECT STATEMENT                    |                           |    1 |   459 |   26 | 00:00:01 |
|    1 |   SORT ORDER BY                     |                           |    1 |   459 |   26 | 00:00:01 |
|    2 |    NESTED LOOPS OUTER               |                           |    1 |   459 |   25 | 00:00:01 |
|  * 3 |     HASH JOIN OUTER                 |                           |    1 |   433 |   20 | 00:00:01 |
|    4 |      NESTED LOOPS OUTER             |                           |    1 |   304 |   15 | 00:00:01 |
|    5 |       NESTED LOOPS                  |                           |    1 |   264 |   11 | 00:00:01 |
|    6 |        NESTED LOOPS OUTER           |                           |    1 |   219 |    8 | 00:00:01 |
|  * 7 |         TABLE ACCESS BY INDEX ROWID | XXXX_VISITOR              |    1 |   177 |    4 | 00:00:01 |
|  * 8 |          INDEX RANGE SCAN           | IND_VISITOR_TIME          |    1 |       |    3 | 00:00:01 |
|    9 |         TABLE ACCESS BY INDEX ROWID | XXXX_WX_CUSTOMER          |    1 |    42 |    4 | 00:00:01 |
| * 10 |          INDEX RANGE SCAN           | INDEX_CUS_OPENID          |    1 |       |    2 | 00:00:01 |
| * 11 |        TABLE ACCESS BY INDEX ROWID  | XXXX_BIZ                  |    1 |    45 |    3 | 00:00:01 |
| * 12 |         INDEX RANGE SCAN            | IND_BIZ_CLI               |    1 |       |    2 | 00:00:01 |
|   13 |       TABLE ACCESS BY INDEX ROWID   | XXXX_SATISFACTION         |    1 |    40 |    4 | 00:00:01 |
| * 14 |        INDEX RANGE SCAN             | WS_CUSTOMER_SESSION_ID    |    1 |       |    2 | 00:00:01 |
|   15 |      VIEW                           |                           |    1 |   129 |    5 | 00:00:01 |
|   16 |       HASH GROUP BY                 |                           |    1 |   133 |    5 | 00:00:01 |
| * 17 |        TABLE ACCESS BY INDEX ROWID  | XXXX_ANSWER               |    1 |   133 |    4 | 00:00:01 |
| * 18 |         INDEX RANGE SCAN            | IND_ANSWER_CTIME          |    1 |       |    3 | 00:00:01 |
|   19 |     VIEW PUSHED PREDICATE           |                           |    1 |    26 |    5 | 00:00:01 |
|   20 |      SORT GROUP BY                  |                           |    1 |    59 |    5 | 00:00:01 |
|   21 |       TABLE ACCESS BY INDEX ROWID   | XXXX_VISITOR_DISCONNECT   |    1 |    59 |    5 | 00:00:01 |
| * 22 |        INDEX RANGE SCAN             | IND_WECHATDIS_RECONN      |    1 |       |    3 | 00:00:01 |
-----------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 3 - access("T"."CLIENT_ID"="A"."CLIENT_ID"(+))
* 7 - filter("T"."RECONN"='0' AND "T"."USER_TYPE"='user')
* 8 - access("T"."CREATE_TIME">='2020-06-03 00:00:00' AND "T"."CREATE_TIME"<='2020-06-03 23:59:59')
* 10 - access("T"."OPEN_ID"="C"."OPEN_ID"(+))
* 11 - filter("B"."SKILL" IS NOT NULL)
* 12 - access("T"."CLIENT_ID"="B"."CLIENT_ID")
* 14 - access("S"."CUSTOMER_SESSION_ID"(+)="B"."CLIENT_ID")
* 17 - filter(TO_NUMBER("VISITOR_MSG_COUNT")>0 AND TO_NUMBER("AGENT_MSG_COUNT")>0)
* 18 - access("CREATE_TIME">='2020-06-03 00:00:00' AND "CREATE_TIME"<='2020-06-03 23:59:59')
* 22 - access("D"."RECONN_FIRST_ID"="T"."CLIENT_ID")
```



### 效果对比

改写前：
![01.png](http://cdn.lifemini.cn/dbblog/20200911/969b36ae71184f4798ce428485865531.png)

改写后：
![02.png](http://cdn.lifemini.cn/dbblog/20200911/a8b6198feef04a799ec24f5761f0ecfd.png)

在该子查询中，将外层查询的谓词过滤条件调整到内层子查询中。

改变前：

```
select reconn_first_id,
                    client_id,
                    max(agent_id) agent_id,
                    max(answer_time) answer_time,
                    sum(agent_msg_count) as agent_msg_count,
                    sum(visitor_msg_count) as visitor_msg_count
               from XXXX_answer
              where CREATE_TIME >= '2020-06-03 00:00:00'
                and CREATE_TIME <= '2020-06-03 23:59:59'
              group by reconn_first_id, client_id
```

改变后：

```
select reconn_first_id,
                    client_id,
                    max(agent_id) agent_id,
                    max(answer_time) answer_time,
                    sum(agent_msg_count) as agent_msg_count,
                    sum(visitor_msg_count) as visitor_msg_count
               from XXXX_answer
              where CREATE_TIME >= '2020-06-03 00:00:00'
                and CREATE_TIME <= '2020-06-03 23:59:59'
                and visitor_msg_count > 0
                and agent_msg_count > 0
              group by reconn_first_id, client_id
access("D"."RECONN_FIRST_ID"="T"."CLIENT_ID")
```

在改变之后，增加了如下顾虑信息。

```
* 17 - filter(TO_NUMBER("VISITOR_MSG_COUNT")>0 AND TO_NUMBER("AGENT_MSG_COUNT")>0)
* 18 - access("CREATE_TIME">='2020-06-03 00:00:00' AND "CREATE_TIME"<='2020-06-03 23:59:59')
```



### 改写原理

在左连接之前减小结果集的返回，达到优化的目的。