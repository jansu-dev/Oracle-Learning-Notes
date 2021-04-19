---
title:[Oracle]--RAC重建UNDO表空间
date:2020-06-17
---



对于rac环境如何重建undo表空间。



#### 准备工作

重建之前，备份参数文件以防数据库出现问题无法启动。

```
create pfile='/home/oracle/bak_pfile_20200617.ora' from spfile;
```

最好，操作之前对数据库全备份。

```
rman target /

backup database;

list backup;
```



#### 节点一

操作步骤如下：

```
create undo tablespace undo1_tmp datafile '+DG_DATA' size 10g autoextend on next 2g;

show parameter undo_tablespace;

alter system set undo_tablespace='UNDO1_TMP' SCOPE=BOTH;


-----------查看是否切换完毕-----
show parameter undo_tablespace;

-----------查看是否切换完毕-----


drop tablespace UNDOTBS1 including contents and datafiles;

create undo tablespace UNDOTBS1 datafile '+DG_ORA' size 10g autoextend on next 2g maxsize 30G;

show parameter undo_tablespace;

alter system set undo_tablespace='UNDOTBS1' SCOPE=BOTH;

show parameter undo_tablespace;

drop tablespace undo1_tmp including contents and datafiles;

select tablespace_name,file_name from dba_data_files where tablespace_name='UNDOTBS1';
```



### 节点二

如果存在节点二也需重建的情况，参照节点一修改步骤，重建节点二undo表空间即可。