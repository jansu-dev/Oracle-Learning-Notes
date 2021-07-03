---
title:[Oracle]-OGG-01296 Error mapping问题的诊断与解决
date:2020-09-27
---



​		下班后收到脚本发来的报警短信提示，生产库一个OGG进程挂掉了，经过尝试通过跳过事务的方法解决此次问题。



### 问题描述

通过 "view report RT1_2Y(Replicat组名)" 查看日志关键信息如下，首先尝试重启进程但是发现RBA一直没有继续增加，整明当前进程依旧存在问题。果不其然，等待一段时间之后，该进程又挂掉了。

```
ERROR OGG-01296 Error mapping from CP_XXX._POSTING_INFO to CP_XXX.TM.......
```

通过sql查看该进程正在执行的SQL，发现改进程正在执行一条update语句。

```
GGSCI (ssy2015-app-09) 2> exit
ssy2015-app-09:/home/oracle$ ps -ef|grep RT1_2Y
oracle   14323 25944 11 20:41 ?        00:05:32 /lsi/ggs/replicat PARAMFILE /lsi/ggs/dirprm/rt1_2y.prm REPORTFILE /lsi/ggs/dirrpt/RT1_2Y.rpt PROCESSID RT1_2Y USESUBDIRS
oracle   17790 17609  0 21:30 pts/0    00:00:00 grep RT1_2Y


ssy2015-app-09:/home/oracle$ ps -ef|grep 14323
oracle   14323 25944 11 20:41 ?        00:05:32 /lsi/ggs/replicat PARAMFILE /lsi/ggs/dirprm/rt1_2y.prm REPORTFILE /lsi/ggs/dirrpt/RT1_2Y.rpt PROCESSID RT1_2Y USESUBDIRS
oracle   14327 14323 14 20:41 ?        00:06:58 oraclegd1 (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))
oracle   17800 17609  0 21:30 pts/0    00:00:00 grep 14323


select sa.* from v$process pr,v$session ss,v$sqlarea sa 
where pr.addr=ss.paddr and ss.sql_hash_value=sa.hash_value 
and pr.spid=14327
```

查出当前进程的RBA，并使用logdump工具查看当前正在执行的事务信息。这是一个异构的表与表之间的同步，查询在该表中该条记录是存在。

```
Reading forward from RBA 54504938
Logdump 3 >n

2020/09/27 19:18:41.002.383 FieldComp            Len   485 RBA 54504938
Name: CP_XXX.XXX_POSTING_INFO
After  Image:                                             Partition 4   G  s
 0001 0011 0000 000d 3938 3633 3033 3030 3235 3039 | ........986303002509
 3400 0300 1500 0032 3032 302d 3039 2d32 323a 3030 | 4......2020-09-22:00
 3a30 303a 3030 004b 000c 0000 0008 3135 3030 3031 | :00:00.K......150001
 3633 0063 0005 0000 0001 5200 6400 0c00 0000 0830 | 63.c......R.d......0
 3231 3030 3037 3100 6500 1500 0032 3032 302d 3039 | 2100071.e....2020-09
 2d32 373a 3139 3a30 393a 3037 0066 0004 ffff 0000 | -27:19:09:07.f......
 0067 0004 ffff 0000 0068 0015 ffff 3139 3030 2d30 | .g.......h....1900-0


select * from CP_XXX.XXX_POSTING_INFO where XXX_no = '9863030025094';
....目标端同步表中存在该条信息...
```



### 解决办法

目标端存在该表，说明又由于数据不一致的原因导致的update无法完成。因为该表主要用于报表计算，事务属性的作用不强，因此采用跳过该事务的方法恢复进程同步。通过view report RT1_2Y查出哪个队列文件（t1_s149384）出现了问题。

```
./logdump 


open dirdat/t1_s149384


pos RBA 54504938


Reading forward from RBA 54504938
Logdump 3 >n

2020/09/27 19:18:41.002.383 FieldComp            Len   485 RBA 54504938
Name: CP_XXX.XXX_POSTING_INFO
After  Image:                                             Partition 4   G  s
 0001 0011 0000 000d 3938 3633 3033 3030 3235 3039 | ........986303002509
 3400 0300 1500 0032 3032 302d 3039 2d32 323a 3030 | 4......2020-09-22:00
 3a30 303a 3030 004b 000c 0000 0008 3135 3030 3031 | :00:00.K......150001
 3633 0063 0005 0000 0001 5200 6400 0c00 0000 0830 | 63.c......R.d......0
 3231 3030 3037 3100 6500 1500 0032 3032 302d 3039 | 2100071.e....2020-09
 2d32 373a 3139 3a30 393a 3037 0066 0004 ffff 0000 | -27:19:09:07.f......
 0067 0004 ffff 0000 0068 0015 ffff 3139 3030 2d30 | .g.......h....1900-0

Logdump 4 >n

2020/09/27 19:18:41.002.383 FieldComp            Len   310 RBA 54505567
Name: CP_XXX.XXX_POSTING_INFO
After  Image:                                             Partition 4   G  s
 0001 0011 0000 000d 3131 3730 3836 3038 3832 3535 | ........117086088255
 3300 0300 1500 0032 3032 302d 3039 2d32 373a 3030 | 3......2020-09-27:00
 3a30 303a 3030 004b 000c 0000 0008 3531 3034 3031 | :00:00.K......510401
 3031 0063 0005 0000 0001 5200 6400 0c00 0000 0835 | 01.c......R.d......5
 3130 3636 3531 3700 6500 1500 0032 3032 302d 3039 | 1066517.e....2020-09
 2d32 373a 3139 3a30 383a 3238 0066 0004 ffff 0000 | -27:19:08:28.f......
 0067 0004 ffff 0000 0068 0015 ffff 3139 3030 2d30 | .g.......h....1900-0


./ggsci

alter replicat RT1_2Y,extrba 54505567

start RT1_2Y
```

修复之后，该进程RBA继续增加，OGG恢复正常同步。