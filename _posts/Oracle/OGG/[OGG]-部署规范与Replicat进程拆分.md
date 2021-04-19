---
title:[OGG]-部署规范与Replicat进程拆分
date:2020-12-29
---



##### OGG部署规范

- [源端MGR](源端MGR)
- [源端extract](源端extract)
- [源端PUMP](源端PUMP)
- [目标端MGR](目标端MGR)
- [目标端Replicat--1](目标端Replicat--1)
- [目标端Replicat--2](目标端Replicat--2)
- [目标端异构造--1](目标端异构造--1)
- [目标端异构造--2](目标端异构造--2)



### 源端MGR

```
port 7809
dynamicportlist 7800-8000
autorestart extract *,waitminutes 2,resetminutes 5
purgeoldextracts /ggs/dirdat/*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/a1*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/yy*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/vv*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/tt*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/ao*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/ss*, usecheckpoints, minkeepfiles 10
purgeoldextracts /ggs/dirdat/z1*, usecheckpoints, minkeepfiles 10
autostart er *
LAGCRITICALMINUTES 30
LAGINFOMINUTES 10
LAGREPORTMINUTES 5
```



### 源端extract

```
GGSCI (db-waybill-test01) 2> view params EORA1

extract eora1
dynamicresolution
USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBBAVHJIRBNENJRCHCBBWIDALD,ENCRYPTKEY
default
exttrail /ggs/dirdat/a1
EOFDELAYCSECS 30
FETCHOPTIONS FETCHPKUPDATECOLS
FLUSHCSECS 30
NOCOMPRESSDELETES
NOCOMPRESSUPDATES
TRANLOGOPTIONS BUFSIZE 10000000
TRANLOGOPTIONS DBLOGREADER, DBLOGREADERBUFSIZE 4096000
table cp_tms.TMS_MAIL_TRAJECTORY,keycols(WORKSCAN_TIME,WORKSCAN_ID);
```



### 源端PUMP

```
extract P_T1_67
dynamicresolution
passthru
CACHEMGR CACHESIZE 2G
rmthost 10.2.196.67,mgrport 7809,compress
rmttrail /lsi/ggs/dirdat/t1
table cp_tms.TMS_MAIL_POSTING_INFO;
```



### 目标端MGR

```
GGSCI (sgdSSD-24) 2> view params MGR

port 7809
dynamicportlist 7800-8000
autorestart extract *,waitminutes 2,resetminutes 5
purgeoldextracts /lsi/ggs/dirdat/*, usecheckpoints, minkeepfiles 10
autostart er *
LAGCRITICALMINUTES 30
LAGINFOMINUTES 10
LAGREPORTMINUTES 5
```



### 目标端Replicat--1

```
GGSCI (sgdSSD-24) 4> view params RB1_1Y

replicat rb1_1y
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
SETENV (ORACLE_SID = "tmsdb")
USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBBAVHJIRBNENJRCHCBBWIDALD,ENCRYPTKEY
default
assumetargetdefs
handlecollisions
discardfile  /lsi/ggs/rb1_1y.dsc, append, megabytes 100
map cp_tms.TMS_MAIL_POSTING_INFO,target cp_tms.TMS_MAIL_POSTING_INFO,FILTER(@RAN
GE(1,4,MAIL_NO,POSTING_DATE));
```



### 目标端Replicat--2

```
GGSCI (sgdSSD-24) 6> view params RB1_2Y

replicat rb1_2y
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
SETENV (ORACLE_SID = "tmsdb")
USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBBAVHJIRBNENJRCHCBBWIDALD,ENCRYPTKEY
default
assumetargetdefs
handlecollisions
discardfile  /lsi/ggs/rb1_2y.dsc, append, megabytes 100
map cp_tms.TMS_MAIL_POSTING_INFO,target cp_tms.TMS_MAIL_POSTING_INFO,FILTER(@RAN
GE(2,4,MAIL_NO,POSTING_DATE));
```



### 目标端异构造--1

```
replicat rt2_1x
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
SETENV (ORACLE_SID = "tmsdb")
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBBAVHJIRBNENJRCHCBBWIDALD,ENCRYPTKEY
default
assumetargetdefs
handlecollisions
SOURCEDEFS /lsi/ggs/dirdef/tms_mail_monitor.def
discardfile  /lsi/ggs/rt2_1x.dsc, append, megabytes 100
map cp_tms.TMS_MAIL_MONITOR_3,target cp_tms.TMS_MAIL_MONITOR_3,
colmap (
MAIL_NO = MAIL_NO,
ACTION_TYPE = ACTION_TYPE,
PLAN_AGENCY = PLAN_AGENCY,
PLAN_FREQUENCY = PLAN_FREQUENCY,
PLAN_ACTION_TIME = PLAN_ACTION_TIME,
ACTUAL_AGENCY = ACTUAL_AGENCY,
ACTUAL_FREQUENCY = ACTUAL_FREQUENCY,
ACTUAL_ACTION_TIME = ACTUAL_ACTION_TIME,
ROUTE_CODE = ROUTE_CODE,
ID = ID,
MAIL_BAG_NO = MAIL_BAG_NO,
IN_CITY_NJ = IN_CITY_NJ,
OUT_CUR_PROV_NJ = OUT_CUR_PROV_NJ,
OUT_CUR_CITY_NJ = OUT_CUR_CITY_NJ,
IN_CUR_PROV_NJ = IN_CUR_PROV_NJ,
IN_CUR_CITY_NJ = IN_CUR_CITY_NJ,
UNDLV_NEXT_ACTN_CODE = UNDLV_NEXT_ACTN_CODE,
ALLONTIME_FLAG = ALLONTIME_FLAG,
ROUTE_SEQ_NUM = ROUTE_SEQ_NUM,
),FILTER(@RANGE(1,8,CREATE_TIME,ID));
```



### 目标端构造--2

```
replicat rt2_2x
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
SETENV (ORACLE_SID = "tmsdb")
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
USERID ggs,PASSWORD AACAAAAAAAAAAAPADGQFHHFBBAVHJIRBNENJRCHCBBWIDALD,ENCRYPTKEY
default
assumetargetdefs
handlecollisions
SOURCEDEFS /lsi/ggs/dirdef/tms_mail_monitor.def
discardfile  /lsi/ggs/rt2_2x.dsc, append, megabytes 100
map cp_tms.TMS_MAIL_MONITOR_3,target cp_tms.TMS_MAIL_MONITOR_3,
colmap (
MAIL_NO = MAIL_NO,
ACTION_TYPE = ACTION_TYPE,
PLAN_AGENCY = PLAN_AGENCY,
PLAN_FREQUENCY = PLAN_FREQUENCY,
PLAN_ACTION_TIME = PLAN_ACTION_TIME,
ACTUAL_AGENCY = ACTUAL_AGENCY,
ACTUAL_FREQUENCY = ACTUAL_FREQUENCY,
ACTUAL_ACTION_TIME = ACTUAL_ACTION_TIME,
ROUTE_CODE = ROUTE_CODE,
ID = ID,
MAIL_BAG_NO = MAIL_BAG_NO,
MAIL_BAG_LABEL = MAIL_BAG_LABEL,
OUT_PROV_NJ = OUT_PROV_NJ,
OUT_CITY_NJ = OUT_CITY_NJ,
IN_PROV_NJ = IN_PROV_NJ,
IN_CITY_NJ = IN_CITY_NJ,
OUT_CUR_PROV_NJ = OUT_CUR_PROV_NJ,
OUT_CUR_CITY_NJ = OUT_CUR_CITY_NJ,
IN_CUR_PROV_NJ = IN_CUR_PROV_NJ,
IN_CUR_CITY_NJ = IN_CUR_CITY_NJ,
UNDLV_NEXT_ACTN_CODE = UNDLV_NEXT_ACTN_CODE,
ALLONTIME_FLAG = ALLONTIME_FLAG,
ROUTE_SEQ_NUM = ROUTE_SEQ_NUM,
),FILTER(@RANGE(2,8,CREATE_TIME,ID));
```

