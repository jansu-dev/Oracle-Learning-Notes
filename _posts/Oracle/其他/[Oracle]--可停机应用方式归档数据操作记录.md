---
title:[Oracle]--可停机应用方式归档数据操作记录
date:2020-08-31
---



​		很多公司采用归档数据降低表数据量的方式，改善SQL的运行，如下为一次归档操作的记录，以便以后同模式快速归档数据使用。



### 操作步骤

1. 备份原始表数据
2. 创建相同结构表
3. 新表添加全部索引
4. 向同名新表插入数据
5. 追加权限信息
6. 收集统计信息



### 归档信息

企业信息已处理

需备份的表：
XXX_AGENT_ACTION
XXX_VISITOR
XXX_VISITOR_DISCONNECT
XXX_VISITOR_DAY
XXX_ANSWER
XXX_ANSWER_DAY
XXX_BIZ
XXX_STATUS

1. ​	备份表

   ```
   alter table XXX_AGENT_ACTION rename to XXX_AGENT_ACTION0511;
   alter table XXX_VISITOR rename to XXX_VISITOR0511;
   alter table XXX_VISITOR_DISCONNECT rename to XXX_VISITOR_DISCONNECT0511;
   alter table XXX_VISITOR_DAY rename to XXX_VISITOR_DAY0511;
   alter table XXX_ANSWER rename to XXX_ANSWER0511;
   alter table XXX_ANSWER_DAY rename to XXX_ANSWER_DAY0511;
   alter table XXX_BIZ rename to XXX_BIZ0511;
   alter table XXX_STATUS rename to XXX_STATUS0511;
   ```

2. 创建新表

   ```
   create table XXX_AGENT_ACTION as select  * from XXX_AGENT_ACTION0511 t where t.op_time > '2018-12-31 23:59:59';
   create table XXX_VISITOR as select  * from XXX_VISITOR0511 t where t.create_time > '2018-12-31 23:59:59';
   create table XXX_VISITOR_DISCONNECT as select  * from XXX_VISITOR_DISCONNECT_0511bak t where t.disconnect_time > '2018-12-31 23:59:59';
   create table XXX_VISITOR_DAY as select  * from XXX_VISITOR_DAY0511 t where t.create_time > '2018-12-31 23:59:59';
   create table XXX_ANSWER as select  * from XXX_ANSWER0511 t where t.create_time > '2018-12-31 23:59:59';
   create table XXX_ANSWER_DAY as select  * from XXX_ANSWER_DAY0511 t where t.create_time > '2018-12-31 23:59:59';
   create table XXX_BIZ as select  * from XXX_BIZ0511 t where t.create_time > '2018-12-31 23:59:59';
   create table XXX_STATUS as select  * from XXX_STATUS0511 t where t.create_time > '2018-12-31 23:59:59';
   ```

3. 添加索引

   ```
   alter table XXX_AGENT_ACTION  add primary key (ID);
   CREATE unique INDEX ind_action_id ON XXX_AGENT_ACTION(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_action_action ON XXX_AGENT_ACTION(ACTION);
   CREATE INDEX ind_action_loginid ON XXX_AGENT_ACTION(LOGIN_ID);
   CREATE INDEX ind_action_time ON XXX_AGENT_ACTION(OP_TIME);
   
   alter table XXX_VISITOR  add primary key (ID);
   CREATE INDEX ind_visitor_id ON XXX_VISITOR(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_visitor_uuid ON XXX_VISITOR(PAGE_UUID);
   CREATE INDEX ind_visitor_client ON XXX_VISITOR(CLIENT_ID);
   CREATE INDEX ind_visitor_time ON XXX_VISITOR(CREATE_TIME);
   CREATE INDEX ind_visitor_reconn ON XXX_VISITOR(RECONN_FIRST_ID);
   
   alter table XXX_VISITOR_DISCONNECT add primary key (ID);
   CREATE INDEX ind_wechatdis_id ON XXX_VISITOR_DISCONNECT(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_wechatdis_client ON XXX_VISITOR_DISCONNECT(CLIENT_ID);
   CREATE INDEX ind_wechatdis_dtime ON XXX_VISITOR_DISCONNECT(DISCONNECT_TIME);
   CREATE INDEX ind_wechatdis_reconn ON XXX_VISITOR_DISCONNECT(RECONN_FIRST_ID);
   
   XXX_VISITOR_DAY
   alter table XXX_VISITOR_DAY  add primary key (ID);
   
   CREATE INDEX ind_vday_id ON XXX_VISITOR_DAY(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_vday_client ON XXX_VISITOR_DAY(CLIENT_ID);
   CREATE INDEX ind_vday_dtime ON XXX_VISITOR_DAY(DISCONNECT_TIME);
   CREATE INDEX ind_vday_reconn ON XXX_VISITOR_DAY(RECONN_FIRST_ID);
   
   alter table XXX_ANSWER add primary key (ID);
   CREATE INDEX ind_answer_id ON XXX_ANSWER(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_answer_agent ON XXX_ANSWER(AGENT_ID);
   CREATE INDEX ind_answer_atime ON XXX_ANSWER(ANSWER_TIME);
   CREATE INDEX ind_answer_client ON XXX_ANSWER(CLIENT_ID);
   CREATE INDEX ind_answer_ctime ON XXX_ANSWER(CREATE_TIME);
   CREATE INDEX ind_answer_dtime ON XXX_ANSWER(DISCONNECT_TIME);
   CREATE INDEX ind_answer_ia ON XXX_ANSWER(IS_ANSWER);
   CREATE INDEX ind_answer_op ON XXX_ANSWER(OPEN_ID);
   CREATE INDEX ind_answer_recon ON XXX_ANSWER(RECONN_FIRST_ID);
   
   alter table XXX_ANSWER_DAY add primary key (ID);
   CREATE INDEX ind_id ON XXX_ANSWER_DAY(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_client ON XXX_ANSWER_DAY(CLIENT_ID);
   CREATE INDEX ind_time ON XXX_ANSWER_DAY(CREATE_TIME);
   
   alter table XXX_BIZ add primary key (ID);
   CREATE INDEX ind_biz_id ON XXX_BIZ(ID)ONLINE NOLOGGING;
   CREATE INDEX ind_biz_re ON XXX_BIZ(RECONN);
   CREATE INDEX ind_biz_cli ON XXX_BIZ(CLIENT_ID);
   CREATE INDEX ind_biz_time ON XXX_BIZ(CREATE_TIME);
   
   alter table XXX_STATUS add primary key (ID);
   CREATE INDEX ind_status_uid ON XXX_STATUS(USER_SESSION_ID);
   ```

4. 授权

   ```
   grant select on XXX_AGENT_ACTION to WX_STAT with grant option;
   grant select on XXX_VISITOR to WX_STAT with grant option;
   grant select on XXX_VISITOR_DISCONNECT to WX_STAT with grant option;
   grant select on XXX_ANSWER to WX_STAT with grant option;
   grant select on XXX_BIZ to WX_STAT with grant option;
   grant select on XXX_STATUS to WX_STAT with grant option;
   ```

5. 收集统计信息

   ```
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_AGENT_ACTION');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_AGENT_ACTION');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_VISITOR');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_VISITOR_DISCONNECT');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_VISITOR_DAY');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_ANSWER');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_ANSWER_DAY');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_BIZ');
   end；
   /
   begin
   DBMS_STATS.GATHER_TABLE_STATS('USER_XXX','XXX_STATUS');
   end；
   /
   ```