---
title:[Oracle]--生产库使用RMAN全备份恢复DG报错ORA-01152问题解决
date:2020-09-07
---



[Oracle]--生产库使用RMAN全备份恢复DG报错ORA-01152问题解决

```
Thu Aug 27 18:34:01 2020
alter database open
Signalling error 1152 for datafile 1!
Beginning Standby Crash Recovery.
Serial Media Recovery started
Managed Standby Recovery starting Real Time Apply
Media Recovery Waiting for thread 1 sequence 10697
Thu Aug 27 18:35:01 2020
Standby crash recovery need archive log for thread 1 sequence 10697 to continue.
Please verify that primary database is transporting redo logs to the standby database.
Wait timeout: thread 1 sequence 10697
Standby Crash Recovery aborted due to error 16016.
Errors in file /u01/app/oracle/diag/rdbms/orcl_dg/orcl/trace/orcl_ora_5811.trc:
ORA-16016: archived log for thread 1 sequence# 10697 unavailable
Recovery interrupted!
Completed Standby Crash Recovery.
Signalling error 1152 for datafile 1!
Errors in file /u01/app/oracle/diag/rdbms/orcl_dg/orcl/trace/orcl_ora_5811.trc:
ORA-10458: standby database requires recovery
ORA-01152: file 1 was not restored from a sufficiently old backup 
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'
ORA-10458 signalled during: alter database open...
```