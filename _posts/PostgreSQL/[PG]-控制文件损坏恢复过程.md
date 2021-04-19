---
title：[PG]-控制文件损坏恢复过程
date:2020-11-09
---



##### [PG]-控制文件损坏恢复过程

### 模拟文件损坏

```
[postgres@pg1 ~]$ cd $PGDATA
[postgres@pg1 data]$ cd global/
[postgres@pg1 global]$ ll |grep pg_control 
-rw-------. 1 postgres postgres  8192 Sep 15 14:34 pg_control

[postgres@pg1 global]$ mv pg_control pg_control_bak-20200928
[postgres@pg1 global]$ pg_ctl restart
waiting for server to shut down..... done
server stopped
waiting for server to start....postgres: could not find the database system
Expected to find it in the directory "/usr/local/pg12.2/data",
but could not open file "/usr/local/pg12.2/data/global/pg_control": No such file or directory
 stopped waiting
pg_ctl: could not start server
Examine the log output.
```



### 重建控制文件

```
[postgres@pg1 global]$ cd $PGDATA/pg_wal 
[postgres@pg1 pg_wal]$ ll -lrth
total 32M
-rw-------. 1 postgres postgres 16M Sep 13 04:38 000000010000000000000004
-rw-------. 1 postgres postgres 16M Sep 28 11:31 000000010000000000000003
drwx------. 2 postgres postgres  44 Sep 28 11:31 archive_status
```

-l 000000010000000000000005

```
[postgres@pg1 pg_wal]$ cd $PGDATA/pg_multixact/members 
[postgres@pg1 members]$ ll 
total 8
-rw-------. 1 postgres postgres 8192 Apr 26 13:40 0000
```

-O 0x100000000

```
[postgres@pg1 pg_wal]$ cd $PGDATA/pg_multixact/ offsets 
[postgres@pg1 offsets]$ ll
total 8
-rw-------. 1 postgres postgres 8192 Sep 13 04:43 0000
```

-m 0x00010000,0x00010000

```
[postgres@pg1 offsets]$ cd $PGDATA/pg_xact/
[postgres@pg1 pg_xact]$ ll
total 8
-rw-------. 1 postgres postgres 8192 Sep 15 14:34 0000
```

-x 0x000100000



### 开始恢复

```
[postgres@pg1 pg_xact]$ pg_resetwal -l 000000010000000000000005 -O 0x100000000 -m 0x00010000,0x00010000 -x 0x000100000 -f $PGDATA
pg_resetwal: warning: pg_control exists but is broken or wrong version; ignoring it
Write-ahead log reset
```



### 验证恢复

```
[postgres@pg1 pg_xact]$ cd 
[postgres@pg1 ~]$ pg_ctl start
waiting for server to start....2020-09-28 12:01:07.872 EDT [30477] LOG:  starting PostgreSQL 12.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-39), 64-bit
2020-09-28 12:01:07.876 EDT [30477] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2020-09-28 12:01:07.877 EDT [30477] LOG:  listening on IPv6 address "::", port 5432
2020-09-28 12:01:07.880 EDT [30477] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
.2020-09-28 12:01:10.407 EDT [30478] LOG:  database system was shut down at 2020-09-28 11:57:33 EDT
2020-09-28 12:01:10.420 EDT [30477] LOG:  database system is ready to accept connections
 done
server started
[postgres@pg1 ~]$ psql
psql (12.2)
Type "help" for help.
```