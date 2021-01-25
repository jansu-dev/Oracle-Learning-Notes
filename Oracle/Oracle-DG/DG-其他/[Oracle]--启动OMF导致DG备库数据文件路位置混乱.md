---
title:[Oracle]--启动OMF导致DG备库数据文件路位置混乱
date:2020-08-19
---



Oracle导致DG备库数据文件路位置混乱

```
oracle dataguard 备库参数db_file_name_convert/db_create_file_dest注意事项

congse3359 2016-08-14 06:20:15  68  收藏
    在dataguard的配置中，db_file_name_convert是比较常用的参数。如果备库中参数db_create_file_dest未设置，则数据文件路径转换是由参数db_file_name_convert决定。
但如果参数db_file_name_convert和db_create_file_dest都设置时，要注意：
测试环境：oracle version:11.2.0.4 standby_file_management=AUTO 主备同为文件系统
测试结论：
1.在备库创建时:
    a.通过restore还原的数据文件，如果备份中的数据文件是由手动命名的文件，则备库中数据文件的位置是由备份中的数据文件原位置和参数db_file_name_convert决定。
    b.通过restore还原的数据文件，如果备份中的数据文件是由OMF命名的文件，则备库中数据文件的位置是由参数db_create_file_dest决定。
    c.通过recover生成的数据文件，主库手动命名的文件和OMF命名的文件，备库中数据文件的位置都是由参数db_create_file_dest决定。
3.在备库正常使用时：
    a.MRP进程做日志应用，主库手动命名的文件和OMF命名的文件，备库中数据文件的位置都是由参数db_create_file_dest决定。
3.在备库启动时:
    a.之前创建备库时通过restore还原的数据文件，并且该文件是手动命名的文件，该文件启动时依赖参数db_file_name_convert。如果db_file_name_convert不正确，会报ORA-10458，ORA-01157，ORA-01110错误。
    b.之前创建时通过restore还原的数据文件，并且该文件是OMF命名的文件，该文件启动时不依赖参数db_file_name_convert。
    c.通过recover或MRP进程产生的数据文件，并且该文件是手动命名的文件或OMF命名的文件,该文件启动时不依赖参数db_file_name_convert。

参考文档：Dataguard DB/LOG FILE NAME CONVERT has been set but files are created in a different directory (文档 ID 1348512.1)

















------------备库----------------


SQL> show parameter create

NAME                     TYPE     VALUE
------------------------------------ ----------- ------------------------------
create_bitmap_area_size          integer     8388608
create_stored_outlines             string
db_create_file_dest             string     /u01/data/datafile
db_create_online_log_dest_1         string
db_create_online_log_dest_2         string
db_create_online_log_dest_3         string
db_create_online_log_dest_4         string
db_create_online_log_dest_5         string
SQL> alter system set db_create_file_dest='';

System altered.

SQL> show parameter db_create_file_dest

NAME                     TYPE     VALUE
------------------------------------ ----------- ------------------------------
db_create_file_dest             string







------------主库-----------------

SQL> show parameter create

NAME                     TYPE            VALUE
------------------------------------ ---------------------- ------------------------------
create_bitmap_area_size          integer            8388608
create_stored_outlines             string
db_create_file_dest             string            +DATA1
db_create_online_log_dest_1         string
db_create_online_log_dest_2         string
db_create_online_log_dest_3         string
db_create_online_log_dest_4         string
db_create_online_log_dest_5         string
SQL> 
SQL> 
SQL> 
SQL> select file_name from dba_data_files where tablespace_name='HOLLYKMIDX';

FILE_NAME
----------------------------------------
+DATA1/hollycrm/datafile/hollykmidx.577.
976099287

+DATA1/hollycrm/datafile/hollykmidx.286.
939405323

+DATA1/hollycrm/datafile/hollykmidx.2272
.1001974461

+DATA1/hollycrm/datafile/hollykmidx.1440
.1048708251

FILE_NAME
----------------------------------------


SQL>  select FILE_ID,FILE_NAME,STATUS from dba_data_files where tablespace_name='HOLLYKMIDX';

   FILE_ID FILE_NAME                    STATUS
---------- ---------------------------------------- ------------------
    23 +DATA1/hollycrm/datafile/hollykmidx.577. AVAILABLE
       976099287

    16 +DATA1/hollycrm/datafile/hollykmidx.286. AVAILABLE
       939405323

    35 +DATA1/hollycrm/datafile/hollykmidx.2272 AVAILABLE
       .1001974461

    41 +DATA1/hollycrm/datafile/hollykmidx.1440 AVAILABLE
       .1048708251

   FILE_ID FILE_NAME                    STATUS
---------- ---------------------------------------- ------------------


SQL> create tablespace szp_test datafile '+DATA1' size 500M;

表空间已创建。

SQL> select file_name from dba_data_files where tablespace_name='SZP_TEST';

FILE_NAME
----------------------------------------
+DATA1/hollycrm/datafile/szp_test.2334.1
048848143


SQL> alter system switch logfile;

系统已更改。

SQL> /

系统已更改。

SQL> /
















---------备库------------------
SQL> select file_name from dba_data_files where tablespace_name='SZP_TEST';

FILE_NAME
--------------------------------------------------------------------------------
/u01/data/datafile/szp_test.2334.1048848143

SQL> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@SJZ-DB2 datafile]$ cd /u01/data/datafile
[oracle@SJZ-DB2 datafile]$ ll |grep _test
-rw-r----- 1 oracle oinstall   524296192 Aug 19 10:46 szp_test.2334.1048848143
```