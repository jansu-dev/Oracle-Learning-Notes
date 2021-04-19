---
title:[Oracle]--RAC数据库服务器修改大页内存管理方式操作
date:2020-06-17
---



RAC数据库服务器修改大页内存管理方式。 数据库版本：11.2.0.4.0； 内存参数设置：SGA=40；PGA=10；



## [节点一]

### 修改大页内存管理参数

备份原始参数文件并查询大页信息

```
cp /etc/sysctl.conf /home/oracle/sysctl.conf.bak

cp /etc/security/limits.conf /home/oracle/limits.conf.bak

grep Hugepagesize /proc/meminfo

grep HugePages_Total /proc/meminfo
```

在sysctl.conf文件中追加vm.nr_hugepages参数配置；
参数含义：大页内存管理快表总页面数；
计算方式：大页内存配置数量 / 大页快表页面大小（sga(mb)/Hugepagesize(mb)）
本例：vm.nr_hugepages=41984/2=20992

```
vi /etc/sysctl.conf

vm.nr_hugepages=20992

sysctl -p
```

在limits.conf文件中修改上限参数配置；
参数含义（单位/bit）：

1. soft memlock，软件最大可用内存；
2. hard memlock，硬件最大可用内存；
   计算方式：vm.nr_hugepages * 快表页面大小 * 1024
   本例：memlock 2×20992×1024=42991616

```
vi /etc/security/limits.conf

  oracle soft memlock 42991616‬
  oracle hard memlock 42991616‬

cat /etc/security/limits.conf
```



### [节点一关库关集群]

如果oracle上存在OGG，应先关闭OGG相关进程。

```
./ggsci

stop *

stop mgr
```

执行下面命令后，查看所有资源直至全部OFF。

```
sqlplus / as sysdba

shutdown immediate

crsctl stop crs

crsctl check crs

crs_stat -t -v
```



### [节点一重启服务器]

```
reboot
```

起库后，等待CRS集群件自动启动。

```
crsctl check crs

srvctl status -t

crs_stat -t -v
```

如果失败，手动重启crs集群

```
crsctl stop crs 

 crsctl start crs 

 crsctl check crs

 crs_stat -t -v
```



### [备份参数文件]

集群启动成功后,查看spfile位置并备份。

```
sqlplus / as sysdba

SELECT NAME, VALUE, DISPLAY_VALUE FROM V$PARAMETER WHERE NAME ='spfile';

create pfile='/home/oracle/szp_spfile_RAC20' from spfile='+DATA1/cdcc/spfileXXXX.ora';
```



### [修改内存AMM管理为手动管理]

### [节点一修改参数]

```
ALTER SYSTEM RESET MEMORY_MAX_TARGET SCOPE = SPFILE  sid='XXXX1';

ALTER SYSTEM RESET memory_target SCOPE = SPFILE  sid='XXXX1';

ALTER SYSTEM SET SGA_TARGET =40g SCOPE = SPFILE  sid='XXXX1';

ALTER SYSTEM SET SGA_MAX_SIZE =40g SCOPE = SPFILE sid='XXXX1';

ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 10g SCOPE = SPFILE sid='XXXX1';

startup spfile='+DATA1/XXXX/spfileXXXX.ora'

最后show parameter验证。。
```



## [节点二]

### [节点二修改参数]

在节点二上执行操作如下。
之所以在二节点要单独操作，因为ALTER SYSTEM RESET MEMORY_MAX_TARGET SCOPE = SPFILE sid='*';操作是不被oracle允许的，所以要单独操作。

```
ALTER SYSTEM RESET MEMORY_MAX_TARGET SCOPE = SPFILE  sid='XXXX2';

ALTER SYSTEM RESET memory_target SCOPE = SPFILE sid='XXXX2';

ALTER SYSTEM SET SGA_TARGET =40g SCOPE = SPFILE  sid='XXXX2';

ALTER SYSTEM SET SGA_MAX_SIZE =40g SCOPE = SPFILE  sid='XXXX2';

ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 10g SCOPE = SPFILE  sid='XXXX2';

startup spfile='+DATA1/XXXX/spfileXXXX.ora'

最后show parameter验证。
```

至此，RAC修改数据库服务器内存管理方式结束。



## [检查]

Oracle 11.2.0.3及以后版本，可在检查警报日志来验证是否实例是否启用大页。

```
****************** Large Pages Information *****************


 Total Shared Global Region in Large Pages = 28 GB (100%)

 Large Pages used by this instance: 14497 (28 GB)

 Large Pages unused system wide = 1015 (2030 MB) (alloc incr 64 MB)

 Large Pages configured system wide = 19680 (38 GB)

 Large Page size = 2048 KB
```



### <span style='color:red'>[注意]</span>

<span style='color:red'>如果出现一节点启动之后，二节点一直无法open，处于mount状态的情况，观察日志也并没有爆出任何异常的情况，可以先将节点一数据库shutdown，在节点二在起库之后再启动节点一数据库。</span>

<span style='color:red'>具体原因未知，猜测为两节点公用一份参数文件，节点一起库之后存在两节点相互影响的现象，阻止节点二一致无法open。</span>