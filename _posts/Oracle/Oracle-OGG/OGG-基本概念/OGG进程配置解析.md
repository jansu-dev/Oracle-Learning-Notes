### 配置Extract进程拆解  
```
extract eora1
# 配置extract进程名称

dynamicresolution
# 动态

USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBBAVHJIRBNENJRCHCBBWIDALD,ENCRYPTKEY default
# 配置用户登录信息，同时使用密码加密。

exttrail /rmanbackup/ggs/dirdat/a1  
Extract捕获进程将trail文件保存在/rmanbackup/ggs/dirdat/a1路径下

CACHEMGR CACHESIZE 4G
限制OGG为缓存没有提交的事务记录在内存中的空间占用

EOFDELAYCSECS 30
每30ms强制进程读取REDO日志，默认1s，最小10ms

FETCHOPTIONS FETCHPKUPDATECOLS
使用OGG进行数据初始化时，和HANDLECOLLISIONS配合使用，来解决replicat 主键更新丢失的问题
默认1s，最小10ms
FLUSHCSECS 30
缓冲区写满后，指定每隔20ms向trail文件中写入一次，默认1s，最小10ms。

NOCOMPRESSDELETES
COMPRESSDELETES参数是默认值，只记录删除有主键的值，NOCOMPRESSDELETES 参数记录所有列删除值

NOCOMPRESSUPDATES
NOCOMPRESSUPDATES参数是默认值，只记录更新有主键的值，NOCOMPRESSDELETES 参数记录所有列更新值

TRANLOGOPTIONS BUFSIZE 10000000
TRANLOGOPTIONS DBLOGREADER, DBLOGREADERBUFSIZE 4096000
BUFSIZE选项的值必须始终至少等于或大于的值DBLOGREADERBUFSIZE。
BUFSIZE控制从事务日志中读取缓冲区数据的最大大小（单位/字节）。
DBLOGREADERBUFSIZE控制对内部缓冲区的读取操作的最大大小（单位/字节）。
DBLOGREADER用在常规磁盘和原始磁盘上挖掘日志，且可代替直接连接到Oracle ASM实例。数据库系统必须包含此API模块和正在运行的库。读取大小有DBLOGREADERBUFSIZE控制。

table cp_tms.TMS_MAIL_TRAJECTORY,keycols(WORKSCAN_TIME,WORKSCAN_ID);
通过keycols来实现指定目标端以源端哪些列为主键，默认使用所有列当主键。
```



### 配置Replicat进程拆解

```
GGSCI (sgdSSD-24) 2> view params RB1_1Y

replicat rb1_1y
# 配置进程名称，要求不超过八个字符。

SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
SETENV (ORACLE_SID = "tmsdb")
# 配置进程环境设置，这里设置了SID和数据库客户端字符集。

USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBB......,ENCRYPTKEY default  
# 配置用户登录信息，同时使用密码加密。

assumetargetdefs
# 官方文档：State whether or not source and target definitions are identical
# 判断两端数据库类型及结构是都一致。
# 一致时，用ASSUMETARGETDEFS。 
# 不一致时，用SOURCEDEFS数据结构定义文件，实现不同数据库间同步。
# 格式：SOURCEDEFS <full_pathname> | ASSUMETARGETDEFS

handlecollisions
# 目标端存在相同数据时，忽略重复数据错误。

discardfile  /lsi/ggs/rb1_1y.dsc, append, megabytes 100
# 定义discardfile文件位置，如果处理中油记录出错会写入到此文件中。

map cp_tms.TMS_MAIL_POSTING_INFO,
target cp_tms.TMS_MAIL_POSTING_INFO,
FILTER(@RANGE(1,4,MAIL_NO,POSTING_DATE));
# 配置两端映射关系，使用filter过滤数据。
```



### OGG配置过程中部分参数说明  

##### MANAGER进程参数配置说明：
```
PORT：指定服务监听端口；这里以7839为例，默认端口为7809

DYNAMICPORTLIST：动态端口，源端和目标端的Collector、Replicat、GGSCI进程通信也会使用此端口；

COMMENT：注释行，也可以用--来代替;

AUTOSTART：指定在管理进程启动时自动启动哪些进程；

AUTORESTART：自动重启参数设置：时间 次数

PURGEOLDEXTRACTS：定期清理trail文件设置：设置清理天数。

LAGREPORT、LAGINFO、LAGCRITICAL：
定义数据延迟的预警机制：本处设置表示MGR进程每隔1小时检查EXTRACT的延迟情况，如果超过了30分钟就把延迟作为信息记录到错误日志中，如果延迟超过了45分钟，则把它作为警告写到错误日志中。
```

##### EXTRACT进程参数配置说明
```
SETENV：配置系统环境变量

USERID/PASSWORD：指定OGG连接数据库的用户名和密码

TABLE：定义需复制的表需以；结尾

TABLEEXCLUDE：定义排除的表，可用通配符指定排除表

GETUPDATEAFTERS|IGNOREUPDATEAFTERS：是否在队列中写入后影像，缺省复制

GETUPDATEBEFORES| IGNOREUPDATEBEFORES：是否在队列中写入前影像，缺省不复制

GETUPDATES|IGNOREUPDATES：是否复制UPDATE操作，缺省复制  

GETDELETES|IGNOREDELETES：是否复制DELETE操作，缺省复制

GETINSERTS|IGNOREINSERTS：是否复制INSERT操作，缺省复制

GETTRUNCATES|IGNORETRUNDATES：是否复制TRUNCATE操作，缺省不复制；

RMTHOST：指定目标系统MGR进程的端口号、定义是否使用压缩进行传输；

RMTTRAIL：指定写入到目标断的哪个队列；

EXTTRAIL：指定写入到本地的哪个队列；

SQLEXEC：在extract进程运行时首先运行一个SQL语句；

PASSTHRU：禁止extract进程与数据库交互，适用于Data Pump传输进程；

REPORT：定义自动定时报告；

STATOPTIONS：定义每次使用stat时统计数字是否需要重置；

REPORTCOUNT：报告已经处理的记录条数统计数字；

TLTRACE：打开对于数据库日志的跟踪日志；

DISCARDFILE：定义discardfile文件位置，如果处理中出现记录出错会写入到此文件中；

DBOPTIONS：指定对于某种特定数据库所需要的特殊参数；

TRANLOGOPTIONS：指定在解析数据库日志时所需要的特殊参数 
例如：对于裸设备，可能需要加入以下参数 rawdeviceoggset 0 

WARNLONGTRANS：指定对于超过一定时间的长交易可以在gsserr.log里面写入警告信息；
```


##### REPLICAT进程参数配置说明
```
ASSUMETARGETDEFS：假定两端数据结构一致使用此参数；

SOURCEDEFS：假定两端数据结构不一致，使用此参数指定源端的数据结构定义文件。

MAP:用于指定源端与目标端表的映射关系；  

MAPEXCLUDE：用于使用在MAP中使用*匹配时排除掉指定的表；

REPERROR：定义出错以后进程的响应，一般可以定义为两种：
    ABEND，即一旦出现错误即停止复制，此为缺省配置；
    DISCARD，出现错误后继续复制，只是把错误的数据放到discard文件中。
    DISCARDFILE：定义discardfile文件位置，如果处理中油记录出错会写入到此文件中；

SQLEXEC：在进程运行时首先运行一个SQL语句；

GROUPTRANSOPS：将小交易合并成一个大的交易进行提交，减少提交次数，降低系统IO消耗。

MAXTRANSOPS：将大交易拆分，每XX条记录提交一次。
```



### 参考文章   

- [参考文章1--便于理解ENCRYPTKEY密码加密](http://www.traveldba.com/archives/567)