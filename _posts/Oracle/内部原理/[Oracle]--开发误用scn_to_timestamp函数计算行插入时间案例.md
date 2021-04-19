---
title:[Oracle]--开发误用scn_to_timestamp函数计算行插入时间案例
date:2020-06-23
---

​		

​		数据库开发人员反馈业务出现丢数据的现象，经查最终定位问题为开发错把scn_to_timestamp函数计算入库时间。

- [案例背景](案例背景)
- [案例原理](案例原理)
- [原理实验](原理实验)
- [案例总结](案例总结)



### 案例背景

**企业信息已处理**
开发反馈业务系统出现基表丢数据现象，需要DBA进行排查；
于是我在听完他的反馈后，询问了他是如何判定基表丢数据的；
她给了我一条SQL语句。

```
select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss') 入库时间，l.starttime 来点时间 from xxxx_stat.contactinfo l where l.starttime like '2020-06-14%';
```

通过这条SQL查询结果如图显示，“入库时间“和“来电时间“差距非常大。

![1.jpg](http://cdn.lifemini.cn/dbblog/20200910/2a72d184385e4123b98e93b5946ce661.jpg)

将数据行的主键CONTACTID取出，并查询出对应的块号发现相同“入库时间”的数据行所在数据块的块号也是相同的；
例如：观察下图右上角，字段contractid从“913“到“920“都属于一个块内。

换言之：
即使“913“在凌晨3点提交，
而“920“在凌晨7点提交，
那么“913“的scn也会是凌晨7点，
因为这两行数据在同一个数据块内。

这个本身没有些延迟的问题，
如果写逻辑需要判断入库时间的话，
可以增加一个入库时间字段，解决这个问题。
![2.jpg](http://cdn.lifemini.cn/dbblog/20200910/ea86f285409040b0a7caad415d0d024a.jpg)

业务越**不**繁忙，数据块更新的越不频繁，所造成的"入库时间”与“开始时间”造成的差距也就越大。
![3.jpg](http://cdn.lifemini.cn/dbblog/20200910/3fcf1e7e37f840d28c6b146795bf83cb.jpg)



### 案例原理

oracle存储架构中最小单位是块，一个块可以容纳多个行，而scn就记录在块头位置；
使用scn_to_timestamp函数转出的时间，实际上，进行的操作是解析的块头位置的scn，
Oracle 10g 推出了一个新特性，可以基于行记录scn。如果启用这个新特性，再配合scn_to_timestamp函数来作为入库时间的判定就不会出现错误了。
但出于性能考虑，最终开发人员我的推荐下选择了增加入库时间子段来解决这个问题。



### 原理实验

```
SQL> create table t1 (id int,starttime date);

Table created.

SQL> insert into t1 values(1,sysdate);

1 row created.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:08:08:00000000  2020-06-14 20:08:09

SQL> commit;

Commit complete.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09


SQL> alter system checkpoint;

System altered.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09


SQL> alter system flush buffer_cache;

System altered.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09

SQL> insert into t1 values(2,sysdate);

1 row created.

SQL> select * from t1;

    ID STARTTIME
---------- ---------
     1 14-JUN-20
     2 14-JUN-20

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09
2020-06-14 20:10:33:00000000  2020-06-14 20:14:36

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09
2020-06-14 20:10:33:00000000  2020-06-14 20:14:36



-----------------------------虽然没有commit，但是此时在v$session中也不会显示一直执行会话。

SQL> rollback;

Rollback complete.

SQL> select * from t1;

    ID STARTTIME
---------- ---------
     1 14-JUN-20

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09







-----------------------------------------先刷buffer，再提交



SQL> insert into t1 values(3,sysdate);

1 row created.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss')  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME
----------------------------- -------------------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09
2020-06-14 20:10:33:00000000  2020-06-14 20:23:28

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:10:33:00000000  2020-06-14 20:08:09       1
2020-06-14 20:10:33:00000000  2020-06-14 20:23:28       3

SQL> commit;

Commit complete.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:24:19:00000000  2020-06-14 20:08:09       1
2020-06-14 20:24:19:00000000  2020-06-14 20:23:28       3

SQL> alter system checkpoint;

System altered.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:24:19:00000000  2020-06-14 20:08:09       1
2020-06-14 20:24:19:00000000  2020-06-14 20:23:28       3


SQL> select dbms_rowid.rowid_block_number(rowid,'smallfile') block_id from t1;

  BLOCK_ID
----------
       142
       142




-------------------------------------------一直不提交，日志中的scn就没刷新到块中



SQL> insert into t1 values(4,sysdate);

1 row created.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:24:22:00000000  2020-06-14 20:08:09       1
2020-06-14 20:24:22:00000000  2020-06-14 20:23:28       3
2020-06-14 20:24:22:00000000  2020-06-14 20:40:29       4

SQL> select to_char(sysdate,'yyyy-mm-dd hh24:mi:ss') from dual;

TO_CHAR(SYSDATE,'YY
-------------------
2020-06-14 20:40:49

SQL> select dbms_rowid.rowid_block_number(rowid,'smallfile') block_id from t1;

  BLOCK_ID
----------
       142
       142
       142

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;

TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:24:22:00000000  2020-06-14 20:08:09       1
2020-06-14 20:24:22:00000000  2020-06-14 20:23:28       3
2020-06-14 20:24:22:00000000  2020-06-14 20:40:29       4


------------------------------------------------------------------------------------------
-----------------------间隔好久----------------------------------------------------------
-----------------------会话二查看说明未提交---------------------------------------------
------------------------------------------------------------------------------------------

SQL> select * from t1;

    ID STARTTIME
---------- ---------
     1 14-JUN-20
     3 14-JUN-20

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:24:22:00000000  2020-06-14 20:08:09       1
2020-06-14 20:24:22:00000000  2020-06-14 20:23:28       3

SQL> select to_char(sysdate,'yyyy-mm-dd hh24:mi:ss') from dual;


TO_CHAR(SYSDATE,'YY
-------------------
2020-06-14 20:51:02



------------------------------------------------------------------------------------------
-----------------------间隔好久----------------------------------------------------------
-----------------------会话二查看说明未提交---------------------------------------------
------------------------------------------------------------------------------------------


SQL> select to_char(sysdate,'yyyy-mm-dd hh24:mi:ss') from dual;

TO_CHAR(SYSDATE,'YY
-------------------
2020-06-14 20:58:33

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:24:22:00000000  2020-06-14 20:08:09       1
2020-06-14 20:24:22:00000000  2020-06-14 20:23:28       3
2020-06-14 20:24:22:00000000  2020-06-14 20:40:29       4

SQL> commit;

Commit complete.

SQL> select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;


TO_CHAR(SCN_TO_TIMESTAMP(ORA_ TO_CHAR(L.STARTTIME      ID
----------------------------- ------------------- ----------
2020-06-14 20:58:53:00000000  2020-06-14 20:08:09       1
2020-06-14 20:58:53:00000000  2020-06-14 20:23:28       3
2020-06-14 20:58:53:00000000  2020-06-14 20:40:29       4


-------间隔好久

select to_char(sysdate,'yyyy-mm-dd hh24:mi:ss') from dual;

select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;

commit;

select to_char(scn_to_timestamp(ORA_ROWSCN),'yyyy-mm-dd hh24:mi:ss:ff8') ,to_char(l.starttime,'yyyy-mm-dd hh24:mi:ss'),l.id  from t1 l;
```



### 案例总结

开发人员更多注重逻辑实现；
只有开发人员与运维人员配合；
才能将活干的完美漂亮。