---
title:[Mongo]--内存溢出触发系统oomKiller数据库宕机情况处理
date:2020-07-20
---



##### mongo内存溢出触发系统oomKiller数据库宕机，问题定位。



### 事故现象

### 定位问题

#### 事故时间系统日志

```
Jul 15 23:46:05 SJZ-MONGO kernel: mongod invoked oom-killer: gfp_mask=0xd0, order=0, oom_adj=0, oom_score_adj=0
Jul 15 23:46:05 SJZ-MONGO kernel: mongod cpuset=/ mems_allowed=0-1
Jul 15 23:46:05 SJZ-MONGO kernel: Pid: 194522, comm: mongod Not tainted 2.6.32-573.el6.x86_64 #1
Jul 15 23:46:05 SJZ-MONGO kernel: Call Trace:
Jul 15 23:46:05 SJZ-MONGO kernel: [<ffffffff810d6dd1>] ? cpuset_print_task_mems_allowed+0x91/0xb0
Jul 15 23:46:05 SJZ-MONGO kernel: [<ffffffff8112a5d0>] ? dump_header+0x90/0x1b0
......
......
Jul 15 23:46:05 SJZ-MONGO kernel: [72512]    89 72512    20236       16   4       0             0 pickup
Jul 15 23:46:05 SJZ-MONGO kernel: Out of memory: Kill process 194515 (mongod) score 951 or sacrifice child
Jul 15 23:46:05 SJZ-MONGO kernel: Killed process 194515, UID 0, (mongod) total-vm:656268160kB, anon-rss:15740068kB, file-rss:308kB
```

#### 事故时间sar日志

```
[root@MONGO sa]# sar -r -s 23:00:00 -e 23:59:00 -f /var/log/sa/sa15
Linux 2.6.32-573.el6.x86_64 (SJZ-MONGO)     07/15/2020     _x86_64_    (32 CPU)

11:00:01 PM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit
11:10:01 PM   3296556  12919308     79.67    180108    987752  16796504     68.86
11:20:02 PM   3294408  12921456     79.68    180112    987768  16796872     68.86
11:30:01 PM   3293832  12922032     79.69    180120    987984  16796980     68.86
11:40:01 PM    162148  16053716     99.00      1612    302848  22608192     92.69
11:50:01 PM  15912960    302904      1.87      1020      7624    129052      0.53
Average:      5191981  11023883     67.98    108594    654795  14625520     59.96
```

#### 事故时间mongo日志

```
Wed Jul 15 23:40:08.615 [conn1405] query XXX_cc3.MonitorParam query: { query: {}, orderby: { value: -1 } } ntoreturn:1 ntoskip:0 nscanned:1 scanAndOrde
r:1 keyUpdates:0 locks(micros) r:1183176 nreturned:1 reslen:94 1183ms
Wed Jul 15 23:40:11.427 [conn11] build index XXX_cc3.MonitorLiveData20200715234421402 { _id: 1 }
Wed Jul 15 23:40:11.428 [conn11] build index done.  scanned 0 total records. 0 secs
Wed Jul 15 23:40:14.842 [conn1405] query XXX_cc3.MonitorParam query: { query: {}, orderby: { value: -1 } } ntoreturn:1 ntoskip:0 nscanned:1 scanAndOrde
r:1 keyUpdates:0 locks(micros) r:1457701 nreturned:1 reslen:94 1457ms
Wed Jul 15 23:40:25.711 [conn1405] query XXX_cc3.MonitorParam query: { query: {}, orderby: { value: -1 } } ntoreturn:1 ntoskip:0 nscanned:1 scanAndOrde
r:1 keyUpdates:0 locks(micros) r:1963383 nreturned:1 reslen:94 1963ms
Wed Jul 15 23:40:25.711 [conn41] command XXX_cc3.$cmd command: { count: "CoreAgentStatus", query: {} } ntoreturn:1 keyUpdates:0 locks(micros) r:1592345
 reslen:48 1592ms
Wed Jul 15 23:40:32.287 [conn27] query XXX_cc3.CoreParam query: { key: "issigninagent" } ntoreturn:1 ntoskip:0 nscanned:80 keyUpdates:0 locks(micros) r
:2402110 nreturned:1 reslen:247 2402ms
......
Wed Jul 15 23:41:18.322 [conn41] command XXX_cc3.$cmd command: { count: "CoreSkillGroupStatus", query: {} } ntoreturn:1 keyUpdates:0 locks(micros) r:17
34042 reslen:48 1734ms
......
......
Wed Jul 15 23:44:24.931 [conn33] update XXX_cc3.CoreTaskLog query: { _id: "17844bfb-f903-44c9-be4f-eb5da1190b4a" } update: { _id: "17844bfb-f903-44c9-b
e4f-eb5da1190b4a", className: "models.scheduling.TaskLog", executeMachine: "localhost.localdomain", executeClass: "utils.job.AdvanceTaskJob", executeTi
me: "2020-07-15 23:44:05", finishTime: "2020-07-15 23:44:05" } idhack:1 nupdated:1 upsert:1 keyUpdates:0 locks(micros) w:22552464 22552ms
```



### 解决问题

### 引用文章