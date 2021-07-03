---
title:[Oracle]--因服务器宕机删除OGG进程的恢复工作
date:2020-09-03
---



​		下午，收集突然收到短信显示OGG进程挂了。查看发现，源端ping不同目标端服务器，为防止ogg持续报错先将pump进程删除，日后再进行修复。



### 修复步骤

源端

```
ADD EXTRACT p_s1_112, EXTTRAILSOURCE /rmanbackup/ggs/dirdat/s1

ADD RMTTRAIL /lsi/ggs/dirdat/s1, EXTRACT p_s1_112, MEGABYTES 100

alter extract p_s1_112,begin now
```



### 案例总结

<span style='color:red'>**注意**：不要忘记使pump进程从当前开始应用</span>