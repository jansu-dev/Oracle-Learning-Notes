---
title:[SQL Server]-DBA常用语句
date:2020-07-21
---



#### sqlserver常用语句

T-SQL

- [基本信息](基本信息)
- [存储结构](存储结构)
- [性能信息](性能信息)
- [查询逻辑读取最高的sql](查询逻辑读取最高的sql)
- [查询索引碎片](查询索引碎片)
- [修改索引填充因子](修改索引填充因子)
- [查询未使用过的索引](查询未使用过的索引)
- [查询表下索引使用情况](查询表下索引使用情况)
- [数据文件和事务日志文件信息](数据文件和事务日志文件信息)
- [当前数据库的文件的IO信息](当前数据库的文件的IO信息)
- [查询执行次数最多的100个SQL（MSSQL2012适用）](查询执行次数最多的100个SQL（MSSQL2012适用）)
- [查询执行次数最多的100个存储过程](查询执行次数最多的100个存储过程)
- [查询平均执行时间最长的25个存储过程](查询平均执行时间最长的25个存储过程)
- [查询逻辑读最多的25个存储过程](查询逻辑读最多的25个存储过程)
- [查询物理读最多的25个存储过程](查询物理读最多的25个存储过程)
- [查询逻辑写次数最多的25个存储过程](查询逻辑写次数最多的25个存储过程)
- [查找出可能的无效非聚集索引](查找出可能的无效非聚集索引)
- [服务器磁盘容量和挂载信息](服务器磁盘容量和挂载信息)
- [查询当前用户登录数](查询当前用户登录数)
- [查看磁盘剩余空间](查看磁盘剩余空间)
- [数据库恢复模式](数据库恢复模式)
- [查看最近的全备份信息](查看最近的全备份信息)



### 基本信息

查看数据库启动时间

```
select convert(varchar(30),login_time,120)
from master..sysprocesses where spid=1
```

##### 查看数据库服务器名

select 'Server Name:'+ltrim(@@servername)

##### 查看数据库实例名

select 'Instance:'+ltrim(@@servicename)

##### 用户管理

查看所有数据库用户所属的角色信息
exec sp_helpsrvrolemember



### 存储架构

##### 数据库磁盘空间使用信息

exec sp_spaceused

##### 日志文件大小及使用情况

dbcc sqlperf(logspace)

##### 查询文件组和文件

select
df.[name],df.physical_name,df.[size],df.growth,
f.[name][filegroup],f.is_default
from sys.database_files df join sys.filegroups f
on df.data_space_id = f.data_space_id



### 性能信息

##### 表的磁盘空间使用信息

select
@@total_read [读取磁盘次数],
@@total_write [写入磁盘次数],
@@total_errors [磁盘写入错误数],
getdate() [当前时间]

##### sql server历史记录查询

```
SELECT     TOP 1000 QS.creation_time, SUBSTRING(ST.text, (QS.statement_start_offset / 2) + 1,
                      ((CASE QS.statement_end_offset WHEN - 1 THEN DATALENGTH(st.text) ELSE QS.statement_end_offset END - QS.statement_start_offset) / 2) + 1)
                      AS statement_text, ST.text, QS.total_worker_time, QS.last_worker_time, QS.max_worker_time, QS.min_worker_time
FROM         sys.dm_exec_query_stats QS CROSS APPLY sys.dm_exec_sql_text(QS.sql_handle) ST
WHERE     QS.creation_time BETWEEN '2017-09-09 10:00:00' AND '2017-09-11 18:00:00' AND ST.text LIKE '%%'
ORDER BY QS.creation_time DESC
```

##### 通过MDW查询某一时段历史数据慢SQL

```
SELECT *
FROM snapshots.active_sessions_and_requests AS asar WITH(NOLOCK)
JOIN snapshots.notable_query_text AS nqt WITH(NOLOCK) ON asar.sql_handle = nqt.sql_handle
JOIN snapshots.notable_query_plan AS nqp WITH(NOLOCK) ON asar.plan_handle=nqp.plan_handle
WHERE asar.collection_time BETWEEN '2013-09-22 10:00:00 +08:00' AND '2013-09-22 22:00:00 +08:00';
```

##### 查看CPU活动及工作情况

select
@@cpu_busy,
@@timeticks [每个时钟周期对应的微秒数],
@@cpu_busy*cast(@@timeticks as float)/1000 [CPU工作时间(秒)],
@@idle*cast(@@timeticks as float)/1000 [CPU空闲时间(秒)],
getdate() [当前时间]

##### 查看SQL Server的实际内存占用

select * from sysperfinfo where counter_name like '%Memory%'

sql server检查索引块是否损坏
sqlserver dbcc checkdb

##### T-SQL提供7种数据类型

精确数据类型
近似数值类型
日期和时间类型
字符串
Unicode字符串
二进制字符串
其他数据类型
创建登录名
CREATE LOGIN shcooper WITH PASSWORD = 'Baz1nga'
GO

#### 为登录名创建数据库用户

use [database_name]
go
CREATE USER test_user FOR LOGIN shcooper;
GO

##### 创建表

exec sp_columns TABLE_NAME
类似于Oracle的dba_table_columns

##### 创建索引

CREATE CLUSTERED INDEX [_dta_index_Student_c_7_981578535__K1] ON [dbo].[Student]
(
[id] ASC
)WITH (SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF) ON [PRIMARY]

##### 存储过程

Create procedure
As
Begin

End
Go

##### 设置事务的隔离级别

SET TRANSACTION ISOLATION LEVEL

##### 删除索引

DROP INDEX tablename.index_name

##### 字段分组列转行

CREATE TABLE t1 ( mid INT, uid VARCHAR(1) )
insert into t1 values (1,'a')
insert into t1 values (1,'b')
insert into t1 values (1,'b')
insert into t1 values (1,'c')
insert into t1 values (1,'d')
insert into t1 values (2,'a')
insert into t1 values (2,'b')
insert into t1 values (2,'c')
insert into t1 values (2,'c')
insert into t1 values (3,'a')
insert into t1 values (3,'b')
insert into t1 values (3,'c')
insert into t1 values (3,'c')
select mid,stuff((select distinct ','+uid from t1 where a.mid=mid for xml path('')),1,1,'') AS items
from t1 a group by mid

--结果
mid items
1 a,b,c,d
2 a,b,c
3 a,b,c

##### 常用函数

获取日期Select datepart(day, getdate()) as currentdate
获取时间差Select datediff(hour, 2015-11-16, 2015-11-11) as

#### T-SQL循环

##### 1、普通循环

Create table Student(id int,name varchar(100),demo varchar(100));
1）循环5次来修改学生表信息
--循环遍历修改记录--
declare
@i int
set @i=0
while @i<10000
begin
update Student set demo = @i+5 where id=@i
set @i=@i +1
end;

##### 2、游标循环（没有事务）

1）根据学生表实际数据循环修改信息
---游标循环遍历--
begin
declare @a int,@error int
declare @temp varchar(50)
set @a=1
set @error=0
--申明游标为Uid
declare order_cursor cursor for (select [Uid] from Student)
--打开游标--
open order_cursor
--开始循环游标变量--
fetch next from order_cursor into @temp
while @@FETCH_STATUS = 0 --返回被 FETCH语句执行的最后游标的状态--
begin
update Student set Age=15+@a,demo=@a where Uid=@temp
set @a=@a+1
set @error= @error + @@ERROR --记录每次运行sql后是否正确，0正确
fetch next from order_cursor into @temp --转到下一个游标，没有会死循环
end
close order_cursor --关闭游标
deallocate order_cursor --释放游标
end
go

##### 3、游标循环（事务）

1）根据实际循环学生表信息
---游标循环遍历--
begin
declare @a int,@error int
declare @temp varchar(50)
set @a=1
set @error=0
begin tran --申明事务
--申明游标为Uid
declare order_cursor cursor
for (select [Uid] from Student)
--打开游标--
open order_cursor
--开始循环游标变量--
fetch next from order_cursor into @temp
while @@FETCH_STATUS = 0 --返回被 FETCH语句执行的最后游标的状态--
begin
update Student set Age=20+@a,demo=@a where Uid=@temp
set @a=@a+1
set @error= @error + @@ERROR --记录每次运行sql后是否正确，0正确
fetch next from order_cursor into @temp --转到下一个游标
end
if @error=0
begin
commit tran --提交事务
end
else
begin
rollback tran --回滚事务
end
close order_cursor --关闭游标
deallocate order_cursor --释放游标
end
Go
常用语句
查询数据库逻辑文件名
USE 数据库名
SELECT FILE_NAME(1)
查询实例服务器级别权限
---授予活动监视器权限
USE master
GO
GRANT VIEW SERVER STATE TO person
GO

##### 查询数据库逻辑文件名（日志）

USE 数据库名
SELECT FILE_NAME(2)

##### 附加数据库

sp_attach_db '数据库名','数据库全路径','数据库日志全路径'
GO
USE 数据库名

--添加一个登录前指定默认数据库
EXEC sp_addlogin '登录名','密码','数据库名'
GO

--处理空登录名(使登录用户和数据库的孤立用户对应起来，在这个用户有对象时用)
sp_change_users_login 'update_one','登录名','登录名'
GO

--修改数据库的逻辑文件名（数据）
Alter DATABASE 数据库名
MODIFY FILE(NAME='老数据库逻辑文件名',NEWNAME='新数据库逻辑文件名')
GO

--修改数据库的逻辑文件名（日志）
Alter DATABASE 数据库名
MODIFY FILE(NAME='老日志逻辑文件名',NEWNAME='新日志逻辑文件名')
GO

可能会用到的操作:
--更改当前数据库名称为dbo的登录名为abc
EXEC sp_changedbowner 'abc'

--删除一个登录
EXEC sp_droplogin '登录名'

--赋予这个登录访问数据库的权限
EXEC sp_adduser '登录名','用户名','db_owner'

#### 常用查询

##### 1.TOP 10 慢SQL
SELECT TOP 10 TEXT AS 'SQL Statement'
,last_execution_time AS 'Last Execution Time'
,(total_logical_reads + total_physical_reads + total_logical_writes) / execution_count AS [Average IO]
,(total_worker_time / execution_count) / 1000000.0 AS [Average CPU Time (sec)]
,(total_elapsed_time / execution_count) / 1000000.0 AS [Average Elapsed Time (sec)]
,execution_count AS "Execution Count"
,qp.query_plan AS "Query Plan"
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY total_elapsed_time / execution_count DESC
规定时间内的慢SQL
declare @sKssj varchar(23),
@sJssj varchar(23)
set @sKssj='2012-05-07 01:35:00.000'
set @sJssj='2012-05-07 15:00:00.000'

SELECT
(total_elapsed_time / execution_count)/1000 N'平均时间ms'
,total_elapsed_time/1000 N'总花费时间ms'
,total_worker_time/1000 N'所用的CPU总时间ms'
,total_physical_reads N'物理读取总次数'
,total_logical_reads/execution_count N'每次逻辑读次数'
,total_logical_reads N'逻辑读取总次数'
,total_logical_writes N'逻辑写入总次数'
,execution_count N'执行次数'
,SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
((CASE statement_end_offset
WHEN -1 THEN DATALENGTH(st.text)
ELSE qs.statement_end_offset END

- qs.statement_start_offset)/2) + 1) N'执行语句'
  ,creation_time N'语句编译时间'
  ,last_execution_time N'上次执行时间'
  FROM
  sys.dm_exec_query_stats AS qs CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) as st
  WHERE
  SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
  ((CASE statement_end_offset
  WHEN -1 THEN DATALENGTH(st.text)
  ELSE qs.statement_end_offset END
- qs.statement_start_offset)/2) + 1) not like '%fetch%'
  and creation_time between @sKssj and @sJssj
  ORDER BY
  total_elapsed_time / execution_count DESC;

##### 2.正在执行的SQL

--活动用户和进程的信息
exec sp_who 'active'

--检查锁
SELECT [Spid] = session_id , ecid ,
[Database] = DB_NAME(sp.dbid) ,[User] = nt_username ,
[Status] = er.status , [Wait] = wait_type ,
[Individual Query] = SUBSTRING(qt.text,
er.statement_start_offset / 2,
(CASE WHEN er.statement_end_offset = -1
THEN LEN(CONVERT(NVARCHAR(MAX), qt.text))* 2
ELSE er.statement_end_offset
END - er.statement_start_offset )/ 2) ,
[Parent Query] = qt.text , Program = program_name ,hostname , nt_domain , start_time
FROM sys.dm_exec_requests er
INNER JOIN sys.sysprocesses sp ON er.session_id = sp.spid
CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) AS qt
WHERE session_id > 50 -- Ignore system spids.
AND session_id NOT IN ( @@SPID ) -- Ignore this current statement.
ORDER BY 1 ,2;
--删除解锁
KILL 1000 --spid

--检查死锁
exec sp_who
exec sp_who2

--检查锁与等待
exec sp_lock

##### 3.批量插入数据

use test
go
Create table Student(id int,name varchar(100),demo varchar(100));
go
declare @i int
set @i=50000;
while @i<30000000
begin
insert into Student values (@i,'n'+CONVERT(varchar(10),@i),'demo'+CONVERT(varchar(10),@i+1));
commit;
set @i=@i +1;
end;
select count(*) from student;
select * from student order by id desc;
select * from student where id=39999;



### 查询逻辑读取最高的sql

SELECT TOP ( 25 )

```
P.name AS [SP Name] ,

    Deps.total_logical_reads AS [TotalLogicalReads] ,

    deps.total_logical_reads / deps.execution_count AS [AvgLogicalReads] ,

    deps.execution_count ,

    ISNULL(deps.execution_count / DATEDIFF(SECOND, deps.cached_time,

                                           GETDATE()), 0) AS [Calls/Second] ,

    deps.total_elapsed_time ,

    deps.total_elapsed_time / deps.execution_count AS [avg_elapsed_time] ,

    deps.cached_time
```

FROM sys.procedures AS p

```
INNER JOIN sys.dm_exec_procedure_stats AS deps ON p.[Object_id] = deps.[Object_id]
```

WHERE deps.Database_id = DB_ID()

ORDER BY deps.total_logical_reads DESC



### 查询索引碎片

--创建变量 指定要查看的表

declare @table_id int

set @table_id=object_id('TableName')

--执行

dbcc showcontig(@table_id)
Logical Scan Fragmentation-逻辑扫描碎片：该百分比应该在0％到10％之间，高了则说明有外部碎片。

Extent Scan Fragmentation-扩展盘区扫描碎片：该百分比应该是0％，高了则说明有外部碎片。

扫描密度［最佳值:实际值］：该百分比应该尽可能靠近100％。低了则说明有外部碎片。



### 修改索引填充因子

（FILLFACTOR：填充因子,ONLINE:ON 重建索引时表仍然可以正常使用）
--修改表下所有索引填充因子

ALTER INDEX ALL ON TableName REBUILD WITH (FILLFACTOR=90,ONLINE=ON)

--修改表下指定索引填充因子

ALTER INDEX indexName ON TableName REBUILD WITH (FILLFACTOR = 80);



### 查询未使用过的索引

SELECT DB_NAME(diu.database_id) AS DatabaseName ,
s.name +'.' +QUOTENAME(o.name)AS TableName,
i.index_id AS IndexID ,
i.name AS IndexName,
CASE WHEN i.is_unique =1 THEN 'UNIQUE INDEX'
ELSE 'NOT UNIQUE INDEX' END AS IS_UNIQUE,
CASE WHEN i.is_disabled=1 THEN 'DISABLE'
ELSE 'ENABLE' END AS IndexStatus,
o.create_date AS IndexCreated,
STATS_DATE(o.object_id,i.index_id) AS StatisticsUpdateDate,
diu.user_seeks AS UserSeek ,
diu.user_scans AS UserScans ,
diu.user_lookups AS UserLookups ,
diu.user_updates AS UserUpdates ,
p.TableRows ,
'DROP INDEX ' + QUOTENAME(i.name)

- ' ON ' + QUOTENAME(s.name) + '.'
- QUOTENAME(OBJECT_NAME(diu.object_id)) +';' AS 'Drop Index Statement'
  FROM sys.dm_db_index_usage_stats diu
  INNER JOIN sys.indexes i ON i.index_id = diu.index_id
  AND diu.object_id = i.object_id
  INNER JOIN sys.objects o ON diu.object_id = o.object_id
  INNER JOIN sys.schemas s ON o.schema_id = s.schema_id
  INNER JOIN ( SELECT SUM(p.rows) TableRows ,
  p.index_id ,p.object_id
  FROM sys.partitions p
  GROUP BY p.index_id ,
  p.object_id) p ON p.index_id = diu.index_id
  AND diu.object_id = p.object_id
  WHERE OBJECTPROPERTY(diu.object_id, 'IsUserTable') = 1
  AND diu.database_id = DB_ID()
  AND i.is_primary_key = 0 --排除主键索引
  AND i.is_unique_constraint = 0 --排除唯一索引
  AND diu.user_updates <> 0 --排除没有数据变化的索引
  AND diu.user_lookups = 0
  AND diu.user_seeks = 0
  AND diu.user_scans = 0
  AND i.name IS NOT NULL --排除那些没有任何索引的堆表
  ORDER BY ( diu.user_seeks + diu.user_scans + diu.user_lookups ) ASC,diu.user_updates DESC;
  GO



### 查询表下索引使用情况

select db_name(database_id) as N'数据库名称',
object_name(a.object_id) as N'表名',
b.name N'索引名称',
user_seeks N'用户索引查找次数',
user_scans N'用户索引扫描次数',
max(last_user_seek) N'最后查找时间',
max(last_user_scan) N'最后扫描时间',
max(rows) as N'表中的行数'
from sys.dm_db_index_usage_stats a join
sys.indexes b
on a.index_id = b.index_id
and a.object_id = b.object_id
join sysindexes c
on c.id = b.object_id
where database_id=db_id('数据库名称') --指定数据库
and object_name(a.object_id) not like 'sys%'
and object_name(a.object_id) like '表名' --指定索引表
and b.name is not null
--and b.name like '索引名' --指定索引名称 可以先使用 sp_help '你的表名' 查看表的结构和所有的索引信息
group by db_name(database_id) ,
object_name(a.object_id),
b.name,
user_seeks ,
user_scans
order by user_seeks,user_scans,object_name(a.object_id)

##### 查询表结构信息

SELECT 表名 = CASE WHEN a.colorder = 1 THEN d.name
ELSE ''
END ,
表说明 = CASE WHEN a.colorder = 1 THEN ISNULL(f.value, '')
ELSE ''
END ,
字段序号 = a.colorder ,
字段名 = a.name ,
标识 = CASE WHEN COLUMNPROPERTY(a.id, a.name, 'IsIdentity') = 1
THEN '√'
ELSE ''
END ,
主键 = CASE WHEN EXISTS ( SELECT 1
FROM sysobjects
WHERE xtype = 'PK'
AND parent_obj = a.id
AND name IN (
SELECT name
FROM sysindexes
WHERE indid IN (
SELECT
indid
FROM sysindexkeys
WHERE id = a.id
AND colid = a.colid ) ) )
THEN '√'
ELSE ''
END ,
类型 = b.name ,
占用字节数 = a.length ,
长度 = COLUMNPROPERTY(a.id, a.name, 'PRECISION') ,
小数位数 = ISNULL(COLUMNPROPERTY(a.id, a.name, 'Scale'), 0) ,
允许空 = CASE WHEN a.isnullable = 1 THEN '√'
ELSE ''
END ,
默认值 = ISNULL(e.text, '') ,
字段说明 = ISNULL(g.[value], '')
FROM syscolumns a
LEFT JOIN systypes b ON a.xusertype = b.xusertype
INNER JOIN sysobjects d ON a.id = d.id
AND d.xtype = 'U'AND d.name <> 'dtproperties'
LEFT JOIN syscomments e ON a.cdefault = e.id
LEFT JOIN sys.extended_properties g ON a.id = G.major_id AND a.colid = g.minor_id
LEFT JOIN sys.extended_properties f ON d.id = f.major_id AND f.minor_id = 0
WHERE d.name = 'TableName' --如果只查询指定表,加上此红色where条件，tablename是要查询的表名；去除红色where条件查询说有的表信息
ORDER BY a.id ,a.colorder；



### 数据文件和事务日志文件信息

SELECT f.name AS [File Name], f.physical_name AS [Physical Name],
CAST((f.size/128.0) AS DECIMAL(15,2)) AS [Total Size in MB],
CAST(f.size/128.0-CAST(FILEPROPERTY(f.name,'SpaceUsed') as int)/128.0 as DECIMAL(19,2)) AS [Available Space In MB],[file_id],
fg.name AS [Filegroup Name]
FROM sys.database_files as f WITH (NOLOCK)
LEFT OUTER JOIN sys.data_spaces AS fg WITH (NOLOCK)
ON f.data_space_id=fg.data_space_id OPTION (RECOMPILE);



### 当前数据库的文件的IO信息

SELECT DB_NAME(DB_ID()) AS [Database Name]
,df.[name] AS [Logical Name]
,vfs.[file_id]
,df.physical_name AS [Physical Name]
,vfs.num_of_reads
,vfs.num_of_writes
,vfs.io_stall_read_ms
,vfs.io_stall_write_ms
,CAST(100. * vfs.io_stall_read_ms / (vfs.io_stall_read_ms + vfs.io_stall_write_ms) AS DECIMAL(10, 1)) AS [IO Stall Reads Pct]
,CAST(100. * vfs.io_stall_write_ms / (vfs.io_stall_write_ms + vfs.io_stall_read_ms) AS DECIMAL(10, 1)) AS [IO Stall Writes Pct]
,(vfs.num_of_reads + vfs.num_of_writes) AS [Writes + Reads]
,CAST(vfs.num_of_bytes_read / 1048576.0 AS DECIMAL(10, 2)) AS [MB Read]
,CAST(vfs.num_of_bytes_written / 1048576.0 AS DECIMAL(10, 2)) AS [MB Written]
,CAST(100. * vfs.num_of_reads / (vfs.num_of_reads + vfs.num_of_writes) AS DECIMAL(10, 1)) AS [# Reads Pct]
,CAST(100. * vfs.num_of_writes / (vfs.num_of_reads + vfs.num_of_writes) AS DECIMAL(10, 1)) AS [# Write Pct]
,CAST(100. * vfs.num_of_bytes_read / (vfs.num_of_bytes_read + vfs.num_of_bytes_written) AS DECIMAL(10, 1)) AS [Read Bytes Pct]
,CAST(100. * vfs.num_of_bytes_written / (vfs.num_of_bytes_read + vfs.num_of_bytes_written) AS DECIMAL(10, 1)) AS [Written Bytes Pct]
FROM sys.dm_io_virtual_file_stats(DB_ID(), NULL) AS vfs
INNER JOIN sys.database_files AS df WITH (NOLOCK)
ON vfs.[file_id] = df.[file_id]
OPTION (RECOMPILE);



### 查询执行次数最多的100个SQL（MSSQL2012适用）

SELECT TOP (100) qs.execution_count,qs.total_rows,qs.last_rows,qs.min_rows,qs.max_rows,
qs.last_elapsed_time, qs.min_elapsed_time, qs.max_elapsed_time,
total_worker_time, total_logical_reads,
SUBSTRING(qt.TEXT,qs.statement_start_offset/2 +1,
(CASE WHEN qs.statement_end_offset = -1
THEN LEN(CONVERT(NVARCHAR(MAX), qt.TEXT)) * 2
ELSE qs.statement_end_offset END - qs.statement_start_offset)/2) AS query_text
FROM sys.dm_exec_query_stats AS qs WITH (NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS qt
WHERE qt.dbid = DB_ID()
ORDER BY qs.execution_count DESC OPTION (RECOMPILE);



### 查询执行次数最多的100个存储过程

SELECT TOP(100) p.name AS [SP Name], qs.execution_count,
ISNULL(qs.execution_count/DATEDIFF(Minute, qs.cached_time, GETDATE()), 0) AS [Calls/
qs.total_worker_time/qs.execution_count AS [AvgWorkerTime], qs.total_worker_time AS
qs.total_elapsed_time, qs.total_elapsed_time/qs.execution_count AS [avg_elapsed_time
qs.cached_time
FROM sys.procedures AS p WITH (NOLOCK)
INNER JOIN sys.dm_exec_procedure_stats AS qs WITH (NOLOCK)
ON p.[object_id] = qs.[object_id]
WHERE qs.database_id = DB_ID()
ORDER BY qs.execution_count DESC OPTION (RECOMPILE);



### 查询平均执行时间最长的25个存储过程

SELECT TOP(25) p.name AS [SP Name], qs.total_elapsed_time/qs.execution_count AS [avg_elapsed_time
qs.total_elapsed_time, qs.execution_count, ISNULL(qs.execution_count/DATEDIFF(Minute
GETDATE()), 0) AS [Calls/Minute], qs.total_worker_time/qs.execution_count AS [AvgWorkerTime
qs.total_worker_time AS [TotalWorkerTime], qs.cached_time
FROM sys.procedures AS p WITH (NOLOCK)
INNER JOIN sys.dm_exec_procedure_stats AS qs WITH (NOLOCK)
ON p.[object_id] = qs.[object_id]
WHERE qs.database_id = DB_ID()
ORDER BY avg_elapsed_time DESC OPTION (RECOMPILE);



### 查询逻辑读最多的25个存储过程

SELECT TOP(25) p.name AS [SP Name], qs.total_logical_reads AS [TotalLogicalReads],
qs.total_logical_reads/qs.execution_count AS [AvgLogicalReads],qs.execution_count,
ISNULL(qs.execution_count/DATEDIFF(Minute, qs.cached_time, GETDATE()), 0) AS [Calls/
qs.total_elapsed_time, qs.total_elapsed_time/qs.execution_count
AS [avg_elapsed_time], qs.cached_time
FROM sys.procedures AS p WITH (NOLOCK)
INNER JOIN sys.dm_exec_procedure_stats AS qs WITH (NOLOCK)
ON p.[object_id] = qs.[object_id]
WHERE qs.database_id = DB_ID()
ORDER BY qs.total_logical_reads DESC OPTION (RECOMPILE);



### 查询物理读最多的25个存储过程

SELECT TOP(25) p.name AS [SP Name],qs.total_physical_reads AS [TotalPhysicalReads],
qs.total_physical_reads/qs.execution_count AS [AvgPhysicalReads], qs.execution_count
qs.total_logical_reads,qs.total_elapsed_time, qs.total_elapsed_time/qs.execution_count
AS [avg_elapsed_time], qs.cached_time
FROM sys.procedures AS p WITH (NOLOCK)
INNER JOIN sys.dm_exec_procedure_stats AS qs WITH (NOLOCK)
ON p.[object_id] = qs.[object_id]
WHERE qs.database_id = DB_ID()
AND qs.total_physical_reads > 0
ORDER BY qs.total_physical_reads DESC, qs.total_logical_reads DESC OPTION (RECOMPILE



### 查询逻辑写次数最多的25个存储过程

SELECT TOP(25) p.name AS [SP Name], qs.total_logical_writes AS [TotalLogicalWrites],
qs.total_logical_writes/qs.execution_count AS [AvgLogicalWrites], qs.execution_count
ISNULL(qs.execution_count/DATEDIFF(Minute, qs.cached_time, GETDATE()), 0) AS [Calls/
qs.total_elapsed_time, qs.total_elapsed_time/qs.execution_count AS [avg_elapsed_time
qs.cached_time
FROM sys.procedures AS p WITH (NOLOCK)
INNER JOIN sys.dm_exec_procedure_stats AS qs WITH (NOLOCK)
ON p.[object_id] = qs.[object_id]
WHERE qs.database_id = DB_ID()
AND qs.total_logical_writes > 0
ORDER BY qs.total_logical_writes DESC OPTION (RECOMPILE);



### 查找出可能的无效非聚集索引

SELECT OBJECT_NAME(s.[object_id]) AS [Table Name], i.name AS [Index Name], i.index_id
i.is_disabled, i.is_hypothetical, i.has_filter, i.fill_factor,
user_updates AS [Total Writes], user_seeks + user_scans + user_lookups AS [Total Reads
user_updates ‐ (user_seeks + user_scans + user_lookups) AS [Difference]
FROM sys.dm_db_index_usage_stats AS s WITH (NOLOCK)
INNER JOIN sys.indexes AS i WITH (NOLOCK)
ON s.[object_id] = i.[object_id]
AND i.index_id = s.index_id
WHERE OBJECTPROPERTY(s.[object_id],'IsUserTable') = 1
AND s.database_id = DB_ID()
AND user_updates > (user_seeks + user_scans + user_lookups)
AND i.index_id > 1
ORDER BY [Difference] DESC, [Total Writes] DESC, [Total Reads] ASC OPTION (RECOMPILE)



### 服务器磁盘容量和挂载信息

SELECT DISTINCT vs.volume_mount_point
,vs.file_system_type
,vs.logical_volume_name
,CONVERT(DECIMAL(18, 2), vs.total_bytes / 1073741824.0) AS [Total Size (GB)]
,CONVERT(DECIMAL(18, 2), vs.available_bytes / 1073741824.0) AS [Available Size (GB)]
,CAST(CAST(vs.available_bytes AS FLOAT) / CAST(vs.total_bytes AS FLOAT) AS DECIMAL(18, 2)) * 100 AS [Space Free %]
FROM sys.master_files AS f WITH (NOLOCK)
CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.[file_id]) AS vs
OPTION (RECOMPILE);



### 查询当前用户登录数

SELECT login_name,Count(0) user_count
FROM Sys.dm_exec_requests dr WITH(nolock)
RIGHT OUTER JOIN Sys.dm_exec_sessions ds WITH(nolock)
ON dr.session_id = ds.session_id
RIGHT OUTER JOIN Sys.dm_exec_connections dc WITH(nolock)
ON ds.session_id = dc.session_id
WHERE ds.session_id > 50
GROUP BY login_name
ORDER BY user_count DESC



### 查看磁盘剩余空间

EXEC master.dbo.xp_fixeddrives

SELECT DISTINCT SUBSTRING(volume_mount_point, 1, 1) AS Volume_mount_point
,total_bytes / 1024 / 1024 AS Total_MB
,available_bytes / 1024 / 1024 AS Available_MB
FROM sys.master_files AS f
CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.file_id);



### 数据库恢复模式

SELECT db.[name] AS [Database Name]
,db.recovery_model_desc AS [Recovery Model]
,db.state_desc
,db.log_reuse_wait_desc AS [Log Reuse Wait Description]
,CONVERT(DECIMAL(18, 2), ls.cntr_value / 1024.0) AS [Log Size (MB)]
,CONVERT(DECIMAL(18, 2), lu.cntr_value / 1024.0) AS [Log Used (MB)]
,CAST(CAST(lu.cntr_value AS FLOAT) / CAST(ls.cntr_value AS FLOAT) AS DECIMAL(18, 2)) * 100 AS [Log Used %]
,db.[compatibility_level] AS [DB Compatibility Level]
,db.page_verify_option_desc AS [Page Verify Option]
,db.is_auto_create_stats_on
,db.is_auto_update_stats_on
,db.is_auto_update_stats_async_on
,db.is_parameterization_forced
,db.snapshot_isolation_state_desc
,db.is_read_committed_snapshot_on
,db.is_auto_close_on
,db.is_auto_shrink_on
,db.target_recovery_time_in_seconds
,db.is_cdc_enabled
FROM sys.databases AS db WITH (NOLOCK)
INNER JOIN sys.dm_os_performance_counters AS lu WITH (NOLOCK) ON db.NAME = lu.instance_name
INNER JOIN sys.dm_os_performance_counters AS ls WITH (NOLOCK) ON db.NAME = ls.instance_name
WHERE lu.counter_name LIKE N'Log File(s) Used Size (KB)%'
AND ls.counter_name LIKE N'Log File(s) Size (KB)%'
AND ls.cntr_value > 0
OPTION (RECOMPILE);



### 查看最近的全备份信息

SELECT TOP (30) bs.machine_name
,bs.server_name
,bs.database_name AS [Database Name]
,bs.recovery_model
,CONVERT(BIGINT, bs.backup_size / 1048576) AS [Uncompressed Backup Size (MB)]
,CONVERT(BIGINT, bs.compressed_backup_size / 1048576) AS [Compressed Backup Size (MB)]
,CONVERT(NUMERIC(20, 2), (CONVERT(FLOAT, bs.backup_size) / CONVERT(FLOAT, bs.compressed_backup_size))) AS [Compression Ratio]
,DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) AS [Backup Elapsed Time (sec)]
,bs.backup_finish_date AS [Backup Finish Date]
FROM msdb.dbo.backupset AS bs WITH (NOLOCK)
WHERE DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) > 0
AND bs.backup_size > 0
AND bs.type = 'D' -- Change to L if you want Log backups
AND database_name = DB_NAME(DB_ID())
ORDER BY bs.backup_finish_date DESC
OPTION (RECOMPILE);
存储架构语句

##### 数据文件

Primary data files(主数据文件)
每个数据库都有一个单独的“主数据文件”,默认以.mdf为扩展名，主数据文件时必须的；
主数据文件包含数据库结构相关信息和数据信息；
在创建数据库时，数据库相关信息不仅存储在master数据库上，也存在与primary data file上。
Secondary dta files(二级数据文件)
一个数据库可以有多个二级数据文件，二级数据文件不是必须的，
因为二级数据文件不包含文件位置等信息。
Transaction logs(事务日志文件)
数据库至少有一个事务日志文件，默认扩展名为ldf。

当一个事务结束时，该事务被标记为已提交，但由于缓存缓存信息处于lazy write状态，所以并不意味着数据的更改已经持久化存储了。
The buffer cache maybe updated,but not necessarily the data file.

##### 检查点触发

①　显示调用checkpoint命令；
②　Recover Interval实例设置周期时间到达；
③　简单模式下的数据库备份；
④　简单模式下的数据库文件结构改变；
⑤　结束数据库引擎。

##### 修改默认文件组

Alter database mydb modify filegroup mydb default;

##### 添加数据文件组：

ALTER DATABASE Credit ADD FILEGROUP filegroup1

##### 为对象指定文件组

Create table test(id int,notes text) on mydb

##### 将对象迁移至指定文件组

Alter table test drop constraint pk_test with(move to data)

##### 添加数据文件：

Use database_name
Go
Alter database database_name add file
(name=N’200901’,
filename=N’C:\Program\200901.ndf’,
Size=5MB,
Maxsize=100MB,
Filegrowth=5MB)
To filegroup filegroup1

#### 函数

##### 创建分区函数

Create partition funtion <分区函数名>(<分区列名>)
As range (left|right)
For values[<分区边界值>[，...n]];
AS RANGE 子句指定在对表中数据进行分区时，数据库引擎按升序从左向右排序的情况下，每个指定的分区边界值属于左侧分区还是右侧分区。
FOR VALUES子句指定分区的边界间隔值。
在CREATE PARTITION FUNCTION语句中不需要指定具体的分区列，分区函数只是定义了分区的方法，此方法具体应用在哪个表的哪个列上，则需要在创建表或索引时指定。

##### 删除分区函数

DROP PARTITION FUNCTION <函数名>

##### 创建分区方案

使用CREATE PARTITION SCHEME 语句
创建分区函数后，必须将其与指定的分区方案向关联。
分区方案用于指定分区对应的文件组，即使多个分区同时位于一个文件组，也需要分别为每个分区定义所属的文件组。
可使用CREATE PARTITION SCHEME 语句创建分区方案，具体语句结构如下：

CREATE PARTITION SCHEME <分区方案名>
AS PARTITION <分区函数名>
TO(文件组名1，文件组名2……)

例如：
CREATE PARTITINO SCHEME Consume2009PartitionScheme1
AS PARTITION Consume2009PartitionFunction1
TO (FileGroup1,FileGroup2,FileGroup3,FileGroup4,FileGroup5,FileGroup6,FileGroup7,FileGroup8,FileGroup9,FileGroup10,FileGroup11,FileGroup12)

##### 删除分区方案

DROP PARTITION SCHEME

##### 备份与恢复

恢复模式类型
简单模式（simple）、完全模式(full)、大容量日志(Bulk-Logged)

##### 完全模式(full)

完全模式是默认的恢复模式
优点：可以恢复到数据库失败或指定时间点。
缺点：需要手动管理，否则十五日志会不断增长，消耗磁盘空间。
清楚十五日志只能通过备份事务日志，或切换至simple模式。

##### 简单模式（simple）

简单模式下，在发生检查点事件时，已提交的事务日志将会从Transaction log中删除。
<span style='color:red'>注意：simple恢复模式保存的事务日志可能是部分的，因此可能存在无法用于数据库恢复到指定时间操作。</span>

##### 大容量日志(Bulk-Logged)

大容量日志模式与完全模式类似，区别在对bulk批处理操作会尽量少的记录。
批处理操作：
1.批处理导入BCP(BULK Copy Import)、Bulk insert、bulk使用openrowset
2.大对象操作（LOB），例如text,ntext,image列上使用writenext或updatetext
3.Select into字句
4.Create index、alter index、alter indexrebuild、dbcc reindex

##### 大容量日志

优点：能有效阻止完全模式下非预期日志的增长，适用于数仓或大量批处理操作数据库。
缺点：恢复困难，一般只能恢复到最后的事务日志备份点。
改变数据库的恢复模式语句
Alter database database_name set recovery bulk_logged;

##### 逻辑备份设备

创建逻辑备份脚本
Exec sp_adddumpdevice @devtype=’type’,@logicalname=’Mydump’,@physicalname=’D:/backup/mydb.bak’
删除逻辑备份脚本
Sp_dropdevice @loicalname=’Mydump’
删除逻辑备份脚本同时删除备份文件
Sp_dropdevice @loicalname=’Mydump’，@devfile=’delfile’
逻辑备份触发方法
Backup database mydb to Mybackup with expiredate=’13/01/2020’
或者
Backup database mydb to Mybackup with retaindays=7
With语句指定过期时间属性
备份集与存储集合
Backup database mydb to mybackup with retaindays=7
Name=’FULL’--备份集名称
Mediname=’ALLBackups’--存储集名称

##### 全备份

不管恢复模式是哪一个，所有的备份都必须要有一个全备份，特别是日志备份和
差异备份，如果没有全备份的话，将无法进行恢复
BACKUP DATABASE mydb to DISK=' D: \Backup\mydb. bak'
注意：上述命令是将数据库备份附加到当前的存在的文件上，如果不存在则创建它，并不会覆盖原有文件。要覆盖同名的备份文件，需要指定INIT。
BACKUP DATABASE mydb to DISK=’ D: \Backup\mydb. bak’WITH INIT

##### 差异备份

使用差异备份，便能缩短恢复时间。
BACKUP DATABASE mydb T0 DISK=’ D: \backup\mydb. dif’WITH DIFFERENTIAL, INIT
进行数据库恢复时，先恢复数据库全备份，再恢复数据库差异备份，最后才恢复日志备份。
差异备份是与上- -次全备份紧密相连的，不管期间有多少次日志备份和差异
备份，差异备份还是会从上一-次全备开始备份。因此，经常会遇到这样的一种情
况,在生产库上需要临时使用数据库时，使用BACKUP DATABASE .. T0 DISK=' ..
进行了一个备份，下一次的差异备份便会以这回的全备为准，如果过后把这个临
时全备删除掉后，后面的差异备份就没用了。

##### 日志备份

Full模式或者大容量恢复模式下，事务日志将会持续增长，直至消耗完所在磁盘。
BACKUP L0G mydb_ log T0 DISK=’ D: \backup\mydb. trn'

BACKUP DATABASE mydb T0 DISK=’D: \data' \mydb. bak’WITH CHECKSUM如果备份过程中，发现了错误，SQL Server会错误信息写入MSDB上的SUSPECT_ PAGE表里面。同时，在默认情况下，备份行为会停止的(STOP_ ON_ ERROR) ，以便管理员排查错误。
但备份过程中的校验和验证还有另外一个选项(CONTINUE_ ON_ ERROR) ,也
就是说，如果发现错误，备份过程并不会中断，而是将错误页信息记录在
MSDB..SUSPECT_ PAGE. 上而已。需要注意的是，SUSPECT_ _PAGE表是有行限制的，
最多只能达到1000行，如果达到了的话，备份同样会失败。
激活校验和验证的话，很明显会影响备份的性能。但还是很有必要的。

##### 完全备份和日志备份语句还支持使用密码属性，如:

BACKUP DATABASE mydb T0 DISK=’D: \mydb. bak’WITH PASSWORD=’mydb'

##### 条带化备份

当磁盘空房间有限时，可以实现不同磁盘的条带化备份。
如果使用条带化备份产生多个物理文件，那么丢失任意一部分备份文件，整个备份将失效。
Backup database mydb to disk=’D:\mydb.bak’.
Disk=’E:\mydb.bak’ with init,checksum,continue_on_error

##### 镜像备份

镜像备份围殴在多个磁盘上保留通过一备份的多份备份。
Backup database mydb to disk=’D:\mydb.bak’
Mirror to Disk=’E:\mydb.bak’ with init,checksum,continue_on_error

##### Copy-only备份

差异备份是在全备份基础上的，如果中间差异备份会被打断
中间产出临时全备份，就会出现数据丢失的情况，为解决此情况sql server2005推出copy-only
Copy-only选项不会打断原先的备份计划。
Backup database mydb to disk=’D:\mydb.bak’with init,checksum,copy_only

##### 备份数据文件

全备份数据文件：Backup database mydb file=’D:\mydb.ndf’to disk=’D:\mydb.bak’
差异备份数据文件：Backup database mydb file=’D:\mydb.ndf’ with differential to disk=’D:\mydb_dif.bak’

##### 备份文件组

Backup database mydb filegroup=’PRIMARY’ to disk=’D:\mydbpri.bak’

##### 不完全备份

SQL Server的不完全备份其实相当于完全备份，
如果某一文件组被设置为了只读，而此文件组又需要仅从一次备份。
可使用如下语句：
首先，停止实例；
其次，Backup database mydb read_write_filegroups to disk=’D:\mydb.bak’

##### 恢复

Restore相当于重新构建整个或部分数据库，无法改变数据库状态，联机与脱机。
Recovery其实是一个按照日志前滚的过程。
默认情况下，restore之后会自动进行recovery，可手动指定是否前滚
Restore database mydb from mydbdevice with recovery;
Restore database mydb from mydbdevice with norecovery;