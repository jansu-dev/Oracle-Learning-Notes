---
title: [Oracle]--数据导入导出方法汇总
date: 2020-06-29
---

​		

导入导出常用语法：

- [sqlloader](sqlloader)		
- [数据泵](数据泵)



### sqlloader

#### sqlloader导入数据

```
vi /home/oracle/dir1/emp1.ctl
load data
infile '/home/oracle/dir1/emp1.dat'
insert              --insert 插入表必须是空表，非空表用append
into table emp1
fields terminated by ','
optionally enclosed by '"'
(empno,ename,sal,deptno)

create table scott.emp1 as select empno,ename,sal,deptno from scott.emp where 1=2; 
cd /home/oracle/dir1
sqlldr scott/scott control=emp1.ctl log=emp1.log

或

vi emp.ctl
load data
infile *
append
into table emp1
fields terminated  by   ','
optionally enclosed by '"'
(empno,ename,sal,deptno)
begindata
7369,SMITH,800,20
7499,ALLEN,1600,30
7521,WARD,1250,30

sqlldr scott/scott control=emp.ctl log=emp.log
```

#### sqlloader导出导入数据表

```
exp scott/scott@prod file=d:empdept1.dmp tables=(emp1,dept1)
imp scott/scott@prod file=d:empdept1.dmp
```

#### sqlloader导出导入其他用户对象

```
exp 'sys/system@prod as sysdba' file=d:sysscott.dmp tables=(scott.emp1,scott.dept1)
imp scott/scott@prod file=d:sysscott.dmp
imp 'sys/system@prod as sysdba' file=d:sysscott.dmp fromuser=scott
exp scott/scott@prod file=d:scott.dmp owner=scott   
所有segment name的表才能导出，注意deferred_segment_creation
imp 'sys/system@prod as sysdba' file=d:scott.dmp fromuser=scott touser=scott
imp 'sys/system@prod as sysdba' file=d:scott.dmp fromuser=scott touser=tim


alter tablespace tb1 read only;
exp '/ as sysdba' tablespaces=tb1 transport_tablespace=y file=d:\exp_tb1.dmp
imp userid=\'/ as sysdba\' tablespaces=tb1 transport_tablespace=y file=/u01/oradata/prod/exp_tb1.dmp datafiles=/u01/oradata/prod/MYTB1.DBF 
select tablespace_name,status from dba_tablespaces;
select * from scott.t1;
alter tablespace tb1 read write;

execute dbms_tts.transport_set_check('TEST');
select * from TRANSPORT_SET_VIOLATIONS;
VIOLATIONS
----------------------------------------------------------------------------
ORA-39907: 索引 SCOTT.EMP1_IDX (在表空间 TEST 中) 指向表 SCOTT.EMP1 (在表空间 USERS 中)。

exp 'sys/system@prod as sysdba' file=d:full.dmp full=y
```

##### 导入用户注意使用拥有sys权限用户操作

```
expdp userid=\'/ as sysdba\'  directory=EXPDP dumpfile=szp_ip13_to_ip190_20200413.dmp job_name=20200413 logfile=szp_ip13_to_ip190_20200413.log schemas=cp_nmc content=metadata_only version=10.2

impdp userid=\'/ as sysdba\' directory=WTY dumpfile=szp_ip13_to_ip190_20200413.dmp job_name=20200413 logfile=szp_ip13_to_ip190_20200413.log remap_tablespace=CP_NMC:HOLLY_CC,CP_DATA:HOLLY_CC remap_schema=CP_NMC:CP_NMC
```



### 数据泵

#### 估算索引创建大小及占用临时表空间大小

```
declare
l_index_ddl varchar(1000);
l_used_bytes number;
l_allocated_bytes number;
begin
dbms_space.create_index_cost(
ddl =>'create index IND_LEE_ZU6 on TMS_MAIL_POSTING_INFO (MAIL_NO, POSTING_PROV, MAIL_KIND_CODE,
 MAILED_SCOPE_CODE, ERROR_FLAG, EFFECTS_ALL_STATE, POSTING_ORGCODE, POSTING_DATE, POSTING_TIME,
  POSTING_CITY, POSTING_COUNTY, RCV_PROV, RCV_CITY, RCV_COUNTY, TRANSPORT_CODE, NATURE_CODE,
   STATUS, DEST, CATEGORY_CODE, DELIVERY_ORGCODE, DELIVERY_TIME, CUR_ACTION, CUR_ORG_CODE,
    CUR_ACTION_TIME, CUR_DIRECTION, SENDER_NAME, SENDER_CODE, CUR_ACTION_DESC) tablespace TMS',
used_bytes=>l_used_bytes,
alloc_bytes=>l_allocated_bytes);
dbms_output.put_line('used ='||l_used_bytes||'bytes'||' allocated= '||l_allocated_bytes||'bytes');
end;
/
```

#### 数据泵导表

```
expdp  username/password  directory=WTY_DUMP dumpfile=posting20191219_ip69_2.dmp job_name=20191219_ip69_2 logfile=posting20191219_ip69_2.log tables=TMS_MAIL_POSTING_INFO query="'where POSTING_DATE>to_date(''2019-12-04 00:00:00'',''yyyy-mm-dd hh24:mi:ss'') '"

impdp  username/password  directory=chenyu_dump  dumpfile=posting20191219_ip69_2.dmp  job_name=20191219_ip69_2 logfile=imp_cp_tms20191219_ip69_2.log 
table_exist_action=replace

expdp scott/scott directory=DATA_PUMP_DIR dumpfile=expdp_scott1.dmp tables=emp1 content=data_only reuse_dumpfiles=y
impdp scott/scott directory=DATA_PUMP_DIR dumpfile=expdp_scott1.dmp tables=emp1 content=data_only
impdp userid=\'/ as sysdba\' directory=data_pump_dir dumpfile=empdept.dmp tables=scott.dept remap_schema=scott:tim
```

#### 数据泵导用户

```
expdp system/oracle directory=DATA_PUMP_DIR dumpfile=scott.dmp schemas=scott
impdp system/oracle directory=DATA_PUMP_DIR dumpfile=scott.dmp remap_schema=scott:tim
这步会重建tim用户，密码继承scott的密码，如果tim用户已存在，会有警告，可忽略。
```

#### 数据泵可传输表空间（适用于大规模数据迁移）

```
expdp '/ as sysdba' directory=dir1 dumpfile=tb1.dmp transport_tablespaces=tb1
impdp userid=\'/ as sysdba\' DIRECTORY=DATA_PUMP_DIR DUMPFILE=’TB1.DMP’ TRANSPORT_DATAFILES='/u01/oradata/prod/MYTB1.DBF'

create public database link system_link connect to system identified by oracle using 'prod';
impdp system/oracle tables=tim.t1 network_link=system_link parallel=2
impdp system/oracle schemas=tim network_link=system_link parallel=2 
impdp system/oracle tablespaces=test network_link=system_link parallel=2
```

#### 导入元数据到文本

```
impdp username/password directory=MATE_DIR dumpfile=mate2020032000.dmp include=PROCEDURE  SQLFILE=coresu_procedure.sql
```

#### like导出表

```
expdp username/password directory=exp  dumpfile=LIKE_TP.dmp logfile=LIKE_TP.log  EXCLUDE=TABLE:\"LIKE\'T_PLATFORM%\'\"
```

#### parfile使用

```
DUMPFILE=exp_szp_bak_20200410.dmp
DIRECTORY=WTY_DUMP
job_name=20200410
logfile=exp_szp_bak_20200410.log
tables=
(
STAT_APEALTICKET_MAIL_NO,
STAT_APEALTICKET_MAIL_NO_SO,
STAT_APPEAL_DISPATCH
)

expdp username/password parfile=szp20200410.txt
```

#### DBLINK导表

```
-- Create table
create table TMS_MAIL_POSTING_INFO_SEC
(
  mail_no        VARCHAR2(20) not null,
  posting_date   DATE not null,
  sender_addr    VARCHAR2(340),
  recipient_addr VARCHAR2(340)
)
tablespace TMS 
partition by range (POSTING_DATE) interval (NUMTODSINTERVAL(1, 'day'))(
  partition P1 values less than (TO_DATE(' 2013-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIAN'))
);



begin
  szp_insert_sec_20200101(to_date('2013-07-18 00:00:00', 'yyyy-mm-dd hh24:mi:ss'),to_date('2020-01-01 00:00:00', 'yyyy-mm-dd hh24:mi:ss'));
end;

-- Create table
create table SZP_PRO_OUTPUT
(
  begindate  DATE,
  change_num INTEGER,
  log_date   DATE,
  end_sum    VARCHAR2(10),
  table_name VARCHAR2(100)
)
tablespace TMS
  pctfree 10
  initrans 1
  maxtrans 255
  storage
  (
    initial 64
    next 1
    minextents 1
    maxextents unlimited
  );


create  procedure szp_insert_sec_20200101(begin_date in date,end_date in date) is

begin_in_date date;
begin
  begin_in_date := begin_date;
  loop
      insert into TMS_MAIL_POSTING_INFO_SEC
        select *
          from TMS_MAIL_POSTING_INFO_SEC@DB135 t
         where t.posting_date >= begin_in_date
           and t.posting_date < begin_in_date + 1;
        commit;
    insert into SZP_PRO_OUTPUT
      (begindate, TABLE_NAME)
    values
      (begin_in_date,'TMS_MAIL_POSTING_INFO_SEC');
    commit;
    ---dbms_output.put_line(begin_date||begin_date||update_num);
    begin_in_date := begin_in_date + 1;
    if begin_in_date >= end_date then
      insert into SZP_PRO_OUTPUT
        (begindate, TABLE_NAME)
      values
        (begin_in_date, 'TMS_MAIL_POSTING_INFO_SEC');
      commit;
      exit;
    end if;
  end loop;
end;
```