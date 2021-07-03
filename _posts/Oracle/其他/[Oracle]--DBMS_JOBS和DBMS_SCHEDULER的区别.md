---
title:[Oracle]--DBMS_JOBS和DBMS_SCHEDULER的区别
date:2020-06-23
---



##### DBMS_JOBS和DBMS_SCHEDULER的区别

### DBMS_JOBS使用

```sql
--使用DBMS_JOB
-- 1.创建测试表
create table test(h1 date);

--2.创建存储过程，向test表中插入一条数据
create or replace procedure pro_test is
begin
insert into test values(sysdate);
COMMIT;
end pro_test;
/

--3.创建job
DECLARE
  job1 varchar2(10);
begin
  job1 := 'test';
  dbms_job.submit(job1,'pro_test;',sysdate,'sysdate+1/1440');
  --每天1440分钟，即一分钟运行test过程一次
end;
/

--4.执行job
begin
dbms_job.run(25);
end;
/

--5.停止job
begin
dbms_job.broken(1,true);
end;
/

--6.启用job
begin
dbms_job.broken(25,false);
end;
/


--7.删除job
begin
dbms_job.remove(24);
end;
/

--8.查询测试表
select to_char(h1,'yyyy-mm-dd HH24:mi:ss') H1 from test;

--9.查询视图
select * from dba_jobs;

--FAILURES列大于0说明JOB运行失败

select job,log_user,last_date,failures  from dba_jobs;
```



### DBMS_SCHEDULER使用

```
--1.创建测试表
create table test(h1 date);

--2.创建存储过程，向test表中插入一条数据
create or replace procedure pro_test is
begin
  insert into test values(sysdate);
  COMMIT;
end pro_test;
/


--3.创建schedule

--在schedule中定义了schedule名称、起止时间、调用间隔等参数。
begin
  dbms_scheduler.create_schedule(schedule_name => 'schedule_test',
  start_date => sysdate,
  repeat_interval => 'FREQ=MINUTELY;INTERVAL=1',
  end_date => sysdate+1,
  comments => 'TEST schedule');
end;
/

--查询创建定时任务信息
select schedule_name,repeat_interval from user_scheduler_schedules;


--删除scheduler
begin
   DBMS_SCHEDULER.DROP_SCHEDULE('SCHEDULE_TEST');
end;


--4.创建program

--在program中定义了程序的类型、具体操作、参数个数等参数

begin
  dbms_scheduler.create_program(program_name => 'program_test',
  program_type => 'PLSQL_BLOCK',
  program_action => 'BEGIN PRO_TEST; END;',
  number_of_arguments => 0,
  enabled => TRUE,
  comments => 'TEST program');
end;
/


--5.创建job
--在job中指定了job_name，以及相关联的program_name、schedule_name等参数。

begin
  dbms_scheduler.create_job(job_name => 'job_test',
  program_name => 'program_test',
  schedule_name => 'schedule_test',
  job_class => 'DEFAULT_JOB_CLASS',
  enabled => true,
  auto_drop => true,
  comments => 'TEST procedure');
end;
/


--6.执行job
begin
  dbms_scheduler.run_job(job_name => 'job_test',use_current_session => false);
end;
/


--7.禁用job
begin
  dbms_scheduler.disable('job_test');
end;
/

--8.启用job
begin
  dbms_scheduler.enable('job_test');
end;
/

--9.删除job
begin
  dbms_scheduler.drop_job('job_test');
end;
/

--10.查询视图
select job_name,enabled,state from user_scheduler_jobs;

--11.强制终止job
begin
  dbms_scheduler.stop_job('JOB_name',force => true);
end;
/
```

RAC将job转移到另一个节点执行

```
exec DBMS_SCHEDULER.SET_ATTRIBUTE ('JOB_name','instance_id','2');
```



### 参考文章

- ##### [引用csdn-Alex许恒的文章](https://blog.csdn.net/xuheng8600/article/details/84861351)