---
title: dbms_scheduler定时任务的使用方法
date: 2019-12-27
author: coresu 
categories: 
tags: 
---

#### 
# 


oracle10g以后，dbms_scheduler包被推荐来创建定时任务。

#### 1. 先创建日志表，用于记录存储过程执行时间及结果
```
create table bak_job_test(date_time date,mark varchar2(200));
```
#### 2. 创建一个存储过程，用于创建表
```
create or replace procedure my_test authid current_user is
  v_count number := 0;
  v_mess varchar2(200) := '';
begin
  select count(1) into v_count from user_tables t where t.TABLE_NAME = 'BAK_JOB_TABLES';
  if  v_count > 0 then
    execute immediate 'drop table bak_job_tables purge';
  end if;
  execute immediate 'create table bak_job_tables as select * from user_tables where 1=2';
  insert into bak_job_test(date_time,mark) values (sysdate,'success');
  exception
    when others then
      v_mess := substr(SQLERRM,0,200);
      insert into bak_job_test(date_time,mark) values(sysdate,v_mess);
end;
```

定义存储过程时加上authid current_user可以在存储过程里面使用当前用户所有角色的权限。


#### 3. 使用dbms_scheduler创建定时任务
使用dbms_scheduler需要具有create job权限  
管理定时任务一些操作需要具有MANAGE SCHEDULER权限   
如：dbms_scheduler.stop_job('my_job_test',true);
```
BEGIN
dbms_scheduler.create_job(job_name      => 'my_job_test',
                        job_type        => 'STORED_PROCEDURE',
                        job_action      => 'my_test',
                        start_date      => sysdate,
                        repeat_interval => 'sysdate + 1/1440',
                        enabled         => TRUE,
                        comments        => 'test');
end;
```
##### dbms_scheduler常用存储过程：

1. dbms_scheduler.run(jobName)                  运行job
2. dbms_scheduler.stop_job(jobName,force)       停止job   
force默认为false，oracle建议false停止失败情况下,使用true，且使用true需要有manage scheduler权限。
3. dbms_scheduler.drop_job(jobName)             删除job
4. dbms_scheduler.enable(jobName)               打开job
5. dbms_scheduler.disable(jobName,force)        禁用job  
force参数用于dependencies。如果TRUE,即使其他对象依赖于它,操作也能成功。


##### dbms_scheduler相关视图
1. user_scheduler_jobs                          查看job信息
2. User_Scheduler_Job_Log                       job日志
3. user_scheduler_job_run_details               job运行日志
4. user_scheduler_running_jobs                  正在运行的job

### 参考文章  
[参考文章1-oracle定时任务dbms_job与dbms_scheduler使用方法](https://blog.csdn.net/w892824196/article/details/101702075)