---
title:[SQL Server]--复制数据库
date:2020-07-07
---



##### sqlserver复制数据库



### 源库

1. 新建一个查询窗口，将如下代码稍作修改便可重复使用；

2. 首先将数据文件和日志文件做全备份；

   - [db01]表示备份db01数据库；
   - C:\DBS\Backup\db01_full.bak表示备份到此路径下。

3. 操作代码

   ```
   USE [master]
   GO
   BACKUP DATABASE [db01] TO DISK=N'C:\DBS\Backup\db01_full.bak' WITH FORMAT,COMPRESSION,INIT
   GO
   BACKUP log [db01] TO DISK=N'C:\DBS\Backup\db01_log.bak' WITH FORMAT ,COMPRESSION
   GO
   ```

### 目标库

1. 创建db02数据库（做壳）

2. [db02]表示想要复制并使用的数据库新名称；

3. 操作代码

   ```
   USE [master]
   GO
   CREATE DATABASE [db02]
   GO
   ```

### 重存数据

1. 向db02数据库导入数据；

   - C:\Program Files~\MSSQL\DATA\db02.mdf表示新库的数据文件路径，仅需修改db02.mdf便可；
   - C:\Program Files~\MSSQL11.MSSQLSERVER\MSSQL\DATA\db02_log.ldf表示新库的日志文件路径，仅需修改db02.mdf便可。

2. 操作代码

   ```
   USE [master]
   GO
   RESTORE DATABASE [db02] FROM  DISK = N'C:\DBS\Backup\db01_full.bak' WITH  FILE = 1, 
   MOVE N'db01' TO N'C:\Program Files\Microsoft SQL Server\MSSQL11.MSSQLSERVER\MSSQL\DATA\db02.mdf', 
   MOVE N'db01_Log' TO N'C:\Program Files\Microsoft SQL Server\MSSQL11.MSSQLSERVER\MSSQL\DATA\db02_log.ldf', 
   NOUNLOAD,NORECOVERY,  REPLACE,  STATS = 5
   GO
   ```

### 前滚日志

1. 开始恢复数据；

   - 注意库名和路径。

2. 操作代码

   ```
   USE [master]
   RESTORE DATABASE [db02] FROM  DISK = N'C:\DBS\Backup\db01_log.bak' WITH  FILE = 1, 
   NOUNLOAD,RECOVERY,REPLACE,STATS = 5
   GO
   ```

### 其他修改

1. 修改数据库的逻辑名；

   - 注意库名和路径。

2. 操作步骤

   ```
   Alter DATABASE [db02] 
   MODIFY FILE(NAME='db01',NEWNAME='db02') 
   GO
   commit
   GO
   Alter DATABASE [db02] 
   MODIFY FILE(NAME='db01_log',NEWNAME='db02_log') 
   GO
   commit
   GO
   ```