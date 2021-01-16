---
title: oracle marerialize view远程已设置自动同步无法自动同步
date: 2019-9-15
categories: Oracle
tags: Oracle常见问题
---


#


检查job_queue_processes是否等于0  
Windows端其中job_queue_process的值要大于0才能实现自动刷新


SQL> show  parameter  job_queue_process;
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
job_queue_processes                  integer     0


有关oracle11g的job_queue_processes参数问题
在一个oracle11g数据库里面新建了一个job，job不会在设定的时间运行。
1.job_queue_processes取值范围为0到1000
2.当设定该值为0的时候则任意方式创建的job都不会运行。
3.当设定该值大于1时，且并行执行job时，至少一个做为协调进程。其总数不会超出job_queue_processes的值。

在同步端数据库执行
alter system set job_queue_processes = 10；

SQL> show  parameter  job_queue_process;
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
job_queue_processes                  integer     10
