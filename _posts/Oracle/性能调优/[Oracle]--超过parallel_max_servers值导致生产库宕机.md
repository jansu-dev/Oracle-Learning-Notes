---
title:[Oracle]--超过parallel_max_servers值导致生产库宕机
date:2020-07-02
---



​		早上，突然开发说有一个项目无法正常运行了，需要我查看一下生产库运行情况。一连发现数据库宕机了，于是立即起库恢复业务，最后追查原因发现原来是一个同步脚本异常运行，数据库并行数量超过并行值，从而导致的宕机



### 追查原因

在alert中发现了一段奇怪的告警信息。

```
Adjusting the default value of parameter parallel_max_servers
from 1280 to 970 due to the value of parameter processes (1000)
Starting ORACLE instance (normal)
```

首先，Adjusting the default value of parameter parallel_max_servers from 1280 to 970 due to the value of parameter processes (1000)引起了我的注意。经了解，是导致数据库宕机的只要原因。
总结来说，Oracle依据操作系统cpu核心数计算最大并行度，往往现在服务器的cpu核心数都非常多，因此计算出来的最大并行度非常大，超过了process可用的最大进程数被动态调整小。
例：计算值1280被动态调小至970。

其次，发现Oracle再该时间点有一个脚本在尝试执行；

```
[oracle@DBOracle trace]$ crontab -l
10 7 * * * /u01/app/oracle/dump/scripts/impdp1.sh
10 13 * * * /u01/app/oracle/dump/scripts/impdp2.sh
30 23 * * * /u01/app/oracle/backup/scripts/rman_db0.sh
......
```

执行的内容是将远程的数据同步过来，查看impdp日志发现，近几天该导入脚本均执行失败。

```
#!/bin/bash
source /home/oracle/.bash_profile
export cur_date=`date +%Y%m%d`
cd /u01/app/oracle/dump
impdp XXXX/XXXX DIRECTORY=my_dir DUMPFILE=expdp_XXX_crm_${cur_date}.dmp LOGFILE=impdp_XXX_crm_$
{cur_date}.log SCHEMAS=XXX_crm  TABLE_EXISTS_ACTION=replacemv expdp_XXX_crm_${cur_date}.dmp /u01/app/oracle/dump/bak1
cd /u01/app/oracle/dump/bak1
gzip *.dmp
find /u01/app/oracle/dump/bak1 -name 'expdp*.dmp.gz' -mtime +3 -exec rm -rf {} \;
find /u01/app/oracle/dump/bak1 -name 'expdp*.dmp' -mtime +3 -exec rm -rf {} \;
find /u01/app/oracle/dump/ -name 'expdp*.log' -mtime +3 -exec rm -rf {} \;
find /u01/app/oracle/dump/ -name 'impdp*.log' -mtime +3 -exec rm -rf {} \
```

在询问原因后,得知该脚本的远程同步库已经中断了同步；
也就是impdp的dmp文件本身就存在问题，所以导入肯定有问题；
综合宕机时间及日志信息，最终将宕机原因定位至parallel_max_server的值超过了process的最大值触发了Oracle的BUG。



### 参考文章

- [参考文章1-eygle的文章](https://www.eygle.com/archives/2007/11/parallel_max_servers.html)
- [参考文章2-CSDN-xionglang7的文章](https://blog.csdn.net/xionglang7/article/details/9346183)
- [参考文章3-博客园-数据与人文](https://www.cnblogs.com/shujuyr/p/13089108.html)