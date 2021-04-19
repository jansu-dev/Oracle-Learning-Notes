---
title:[Oracle]-v$session_longops的发现
date:2020-11-07
---





#### v$session_longops的发现

```
select sid,
       opname,
       time_remaining / 60 remaining,
       elapsed_seconds / 60 elapsed,
       sofar,
       totalwork,
       trunc(sofar / totalwork * 100, 2) || '%' as perwork,
       target,
       start_time
  from v$session_longops
 where sofar != totalwork
   and target like '%TMS_MAIL_MONITOR_INFO%';
```