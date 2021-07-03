---
title:[TiDB]-sysbench的使用与参数讲解
date:2020-12-30
---

[TiDB]-sysbench的使用与参数讲解

- [sysbench是什么](sysbench是什么)

- [sysbench的测试方式](sysbench的测试方式)

- [sysbench部署实践](sysbench部署实践)

- [测试sysbench](测试sysbench)

  

### sysbench是什么

在sysbench主要支持 MySQL,pgsql,oracle 这3种数据库

[**sysbench下载地址**](https://github.com/akopytov/sysbench)



### sysbench的测试方式

1、cpu性能
2、磁盘io性能
3、调度程式性能
4、内存分配及传输速度
5、POSIX线程性能
6、数据库性能(OLTP基准测试)现



### sysbench部署实践

> **sysbench依赖安装**

```
yum -y install  make automake libtool pkgconfig libaio-devel vim-common
```

> **sysbench安装**

```
yum list

yum install sysbench
```

> **验证sysbench**

```
sysbench --version
```



### 测试sysbench

#### **测试cpu**

```
[root@tidb01-41 ~]# sysbench --test=cpu --cpu-max-prime=2000 run



WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Prime numbers limit: 2000

Initializing worker threads...

Threads started!

CPU speed:
    events per second: 11785.51

General statistics:
    total time:                          10.0001s
    total number of events:              117871

Latency (ms):
         min:                                    0.07
         avg:                                    0.08
         max:                                   49.70
         95th percentile:                        0.10
         sum:                                 9936.32

Threads fairness:
    events (avg/stddev):           117871.0000/0.00
    execution time (avg/stddev):   9.9363/0.00
```

#### **测试线程**

```
[root@tidb01-41 ~]# sysbench --test=threads --num-threads=500 --thread-yields=100 --thread-locks=4 run



WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 500
Initializing random number generator from current time


Initializing worker threads...

Threads started!


General statistics:
    total time:                          10.0285s
    total number of events:              278840

Latency (ms):
         min:                                    0.02
         avg:                                   17.95
         max:                                  505.02
         95th percentile:                       99.33
         sum:                              5003869.19

Threads fairness:
    events (avg/stddev):           557.6800/68.79
    execution time (avg/stddev):   10.0077/0.01
```

#### **测试IO**

1，prepare阶段，生成需要的测试文件，完成后会在当前目录下生成很多小文件。

```
[tidb@tidb01-41 sysbench]$ sysbench --test=fileio --num-threads=16 --file-total-size=2G --file-test-mode=rndrw prepare



WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
sysbench 1.0.17 (using system LuaJIT 2.0.4)

128 files, 16384Kb each, 2048Mb total
Creating files for the test...
Extra file open flags: (none)
Creating file test_file.0
Creating file test_file.1

......
......

Creating file test_file.126
Creating file test_file.127
2147483648 bytes written in 43.20 seconds (47.41 MiB/sec).
```

2，run阶段

```
[tidb@tidb01-41 sysbench]$ sysbench --test=fileio --num-threads=20 --file-total-size=2G --file-test-mode=rndrw run



WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 20
Initializing random number generator from current time


Extra file open flags: (none)
128 files, 16MiB each
2GiB total file size
Block size 16KiB
Number of IO requests: 0
Read/Write ratio for combined random IO test: 1.50
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing random r/w test
Initializing worker threads...

Threads started!


File operations:
    reads/s:                      11131.77
    writes/s:                     7420.85
    fsyncs/s:                     23992.09

Throughput:
    read, MiB/s:                  173.93
    written, MiB/s:               115.95

General statistics:
    total time:                          10.0137s
    total number of events:              423515

Latency (ms):
         min:                                    0.00
         avg:                                    0.47
         max:                                  317.80
         95th percentile:                        2.11
         sum:                               199481.94

Threads fairness:
    events (avg/stddev):           21175.7500/1228.41
    execution time (avg/stddev):   9.9741/0.02
```

3，清理测试时生成的文件

```
[tidb@tidb01-41 sysbench]$ sysbench --test=fileio --num-threads=20 --file-total-size=2G --file-test-mode=rndrw cleanup



WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Removing test files...
[tidb@tidb01-41 sysbench]$ ll
total 0
```

#### **测试内存**

```
[tidb@tidb01-41 sysbench]$ sysbench --test=memory --memory-block-size=8k --memory-total-size=1G run



WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Running memory speed test with the following options:
  block size: 8KiB
  total size: 1024MiB
  operation: write
  scope: global

Initializing worker threads...

Threads started!

Total operations: 131072 (1671538.03 per second)

1024.00 MiB transferred (13058.89 MiB/sec)


General statistics:
    total time:                          0.0771s
    total number of events:              131072

Latency (ms):
         min:                                    0.00
         avg:                                    0.00
         max:                                    0.08
         95th percentile:                        0.00
         sum:                                   62.89

Threads fairness:
    events (avg/stddev):           131072.0000/0.00
    execution time (avg/stddev):   0.0629/0.00
```

#### 测试mutex

```
[tidb@tidb01-41 sysbench]$ sysbench --test=mutex --num-threads=100 --mutex-num=1000 --mutex-locks=100000 --mutex-loops=10000 run


WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 100
Initializing random number generator from current time


Initializing worker threads...

Threads started!


General statistics:
    total time:                          8.7429s
    total number of events:              100

Latency (ms):
         min:                                 6190.96
         avg:                                 7891.23
         max:                                 8631.53
         95th percentile:                     8484.79
         sum:                               789122.78

Threads fairness:
    events (avg/stddev):           1.0000/0.00
    execution time (avg/stddev):   7.8912/0.53
```

#### 测试OLTP

1，prepare阶段，生成需要的测试表

```
[tidb@tidb01-41 ~]$ sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=192.168.1.41 --mysql-port=4000 --mysql-db=jan --mysql-user=root --mysql-password= --table_size=500000 --tables=1 --threads=100 --events=100000 --report-interval=10 --time=0 prepare



sysbench 1.0.17 (using system LuaJIT 2.0.4)

Initializing worker threads...

Creating table 'sbtest1'...
Inserting 500000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...



[tidb@tidb01-41 ~]$ mysql -uroot -P4000 -h192.168.1.41



Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 190
Server version: 5.7.25-TiDB-v3.0.1 MySQL Community Server (Apache License 2.0)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.


MySQL [(none)]> use jan


Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [jan]> show tables;
+--------------------+
| Tables_in_jan      |
+--------------------+
| jan_hash_member    |
| jan_range_datetime |
| jan_test           |
| sbtest1            |
+--------------------+
4 rows in set (0.00 sec)
```

2，run阶段

```
[tidb@tidb01-41 ~]$ sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=192.168.1.41 --mysql-port=4000 --mysql-db=jan --mysql-user=root --mysql-password= --table_size=500000 --tables=1  --events=100000 --report-interval=10 --time=0 run
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 1
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 1 tps: 57.69 qps: 1154.08 (r/w/o: 807.84/230.76/115.48) lat (ms,95%): 21.11 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 1 tps: 63.60 qps: 1273.24 (r/w/o: 891.63/254.41/127.20) lat (ms,95%): 20.37 err/s: 0.00 reconn/s: 0.00

......
......

[ 3110s ] thds: 1 tps: 35.60 qps: 713.53 (r/w/o: 499.52/142.81/71.20) lat (ms,95%): 35.59 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            1400000
        write:                           400000
        other:                           200000
        total:                           2000000
    transactions:                        100000 (32.09 per sec.)
    queries:                             2000000 (641.87 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          3115.8749s
    total number of events:              100000

Latency (ms):
         min:                                   13.01
         avg:                                   31.15
         max:                                 1957.17
         95th percentile:                       49.21
         sum:                              3115410.37

Threads fairness:
    events (avg/stddev):           100000.0000/0.00
    execution time (avg/stddev):   3115.4104/0.00
```

3，清理测试时生成的测试表

```
[tidb@tidb01-41 ~]$ sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=192.168.1.41 --mysql-port=4000 --mysql-db=jan --mysql-user=root --mysql-password= --table_size=500000 --tables=1  --events=100000 --report-interval=10 --time=0 cleanup
sysbench 1.0.17 (using system LuaJIT 2.0.4)

Dropping table 'sbtest1'...
```

#### 测试表信息

```
sysbench--num-threads=4 --test=oltp--oltp-reconnect-mode=random--mysql-table-engine=innodb --mysql-host=192.168.200.201 --mysql-db=rep_test --oltp-t
```