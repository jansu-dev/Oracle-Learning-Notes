---
title: recover database中“using backup control file”的使用 
date: 2019-10-12
categories: Oracle
tags: Oracle常见问题
---



# 


1. ##### 在恢复时不加using backup controlfile： 
- recover database 
- recover tablespace 
- recover datafile  

​		以上三种情况，Oracle会以当前controlfile所纪录的SCN为准，利用日志文件（archive log、redo log）  
把datafile中的Oracle block恢复到当前控制文件所记录的scn位置。




2. ##### 在恢复时加using backup controlfile：


当需要把数据恢复到比当前controlfile所纪录的SCN以后的位置时 

- control file是backup controlfile
- controlfile是根据trace create的 

以上二者情况出现时，就需要用using backup controlfile.   
作用：在恢复时不会受controlfile所记录scn的限制。

system表空间单独损坏：

- recover database
-  恢复过程：首先比对system表空间的启动SCN（v$datafile_header）和控制文件中记录的SCN（v$datafile），
- 不一致就从system数据文件的启动SCN开始整体的恢复
- 此时控制文件（scn最新）在数据文件之后，通过控制文件记录的scn恢复数据文件。

------

- ORA-01113: file 1 needs media recovery  
- ORA-01110: data file 1: '/u01/app/oracle/oradata/test/system01.dbf'
-   可能是某种原因致使控制问价scn处于数据文件scn之后，可以使用using backup controlfile解决
-   也可通过查询当时的v$datafile_header和v$datafile中的scn验证。  

非系统表空间单独损坏：

- recover database using backup controlfile

- system表空间完好无损不用恢复，非系统表空间需要恢复，因为system表空间的数据文件是最新完好无损

- 在mount下用recover database using backup

- recover database using backup controlfile后，oracle也是先从system的启动scn开始恢复，在恢复system后，

- alter database open resetlogs开库时，oracle报错说那个非系统表空间需要恢复

- 再recover database using backup controlfile恢复，此时便从坏了的非系统表空间的数据文件的SCN开始整体恢复

  

<span style='color:blue'>经验选择：</span>  
<span style='color:blue'>此时恢复的非系统表空间，由于是非主要表空间，一般可以在开库状态下恢复数据库 ;</span>
<span style='color:blue'>即：使用offline非系统表空间，alter database recover “相应datafile”;</span>


