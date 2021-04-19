---
title:[TiDB]-统计信息搜集与使用
date:2020-12-27
---



#### [TiDB]-统计信息搜集与使用

- [登陆TiDB数据库](登陆TiDB数据库)

- [统计信息设置](统计信息设置)

- [搜集统计信息](搜集统计信息)

- [其他](其他)

  

  



### 登陆TiDB数据库

```
[tidb@tidb01-41 tidb-ansible]$ mysql -u root -h 192.168.1.41 -P 4000
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 211
Server version: 5.7.25-TiDB-v3.0.0 MySQL Community Server (Apache License 2.0)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.




MySQL [(none)]> show variables like '%analyze%';
+------------------------------+-------------+
| Variable_name                | Value       |
+------------------------------+-------------+
| tidb_enable_fast_analyze     | 0           |
| tidb_auto_analyze_ratio      | 0.5         |
| tidb_auto_analyze_end_time   | 23:59 +0000 |
| tidb_auto_analyze_start_time | 00:00 +0000 |
+------------------------------+-------------+
4 rows in set (0.01 sec)



MySQL [jan]> show tables;
+---------------+
| Tables_in_jan |
+---------------+
| jan_test      |
+---------------+
1 row in set (0.00 sec)
```



### 统计信息设置

1.tidb_enable_fast_analyze：表示是否开启统计信息设置；
2.tidb_auto_analyze_ratio：表示当表的数据量改变达到50%的时候，进行统计信息搜集；
3.tidb_auto_analyze_start_time/tidb_auto_analyze_end_time：表示搜集统计信息的时间；

<span style=color:red>注意：</span>
<span style=color:red>如果想要避免在业务高峰时间因为搜集统计信息而导致的性能问题，请修改可以搜集统计信息的时间。</span>

```
MySQL [jan]> show variables like '%analyze%';
+------------------------------+-------------+
| Variable_name                | Value       |
+------------------------------+-------------+
| tidb_enable_fast_analyze     | 0           |
| tidb_auto_analyze_ratio      | 0.5         |
| tidb_auto_analyze_end_time   | 23:59 +0000 |
| tidb_auto_analyze_start_time | 00:00 +0000 |
+------------------------------+-------------+
4 rows in set (0.01 sec)
```



### 搜集统计信息

```
MySQL [jan]> show stats_healthy where db_name='jan';
+---------+------------+----------------+---------+
| Db_name | Table_name | Partition_name | Healthy |
+---------+------------+----------------+---------+
| jan     | jan_test   |                |       0 |
+---------+------------+----------------+---------+
1 row in set (0.00 sec)



MySQL [jan]> show stats_healthy where db_name='jan';
+---------+------------+----------------+---------+
| Db_name | Table_name | Partition_name | Healthy |
+---------+------------+----------------+---------+
| jan     | jan_test   |                |     100 |
+---------+------------+----------------+---------+
1 row in set (0.00 sec)
```



### 其他

```

```

MySQL [(none)]> show variables like '%currency%';
+------------------------------------+-------+
| Variable_name | Value |
+------------------------------------+-------+
| tidb_index_lookup_join_concurrency | 4 |
| tidb_hashagg_final_concurrency | 4 |
| tidb_index_serial_scan_concurrency | 1 |
| tidb_checksum_table_concurrency | 4 |
| tidb_hashagg_partial_concurrency | 4 |
| tidb_index_lookup_concurrency | 4 |
| tidb_distsql_scan_concurrency | 15 |
| thread_concurrency | 10 |
| tidb_projection_concurrency | 4 |
| tidb_hash_join_concurrency | 5 |
| innodb_concurrency_tickets | 5000 |
| innodb_commit_concurrency | 0 |
| tidb_build_stats_concurrency | 4 |
| innodb_thread_concurrency | 0 |
+------------------------------------+-------+
14 rows in set (0.01 sec)

```

```

