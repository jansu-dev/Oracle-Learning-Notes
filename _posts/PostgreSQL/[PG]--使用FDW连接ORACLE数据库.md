---
title:[PG]--使用FDW连接ORACLE数据库
date:2020-08-30
---



##### [PG]--连接ORACLE



### 下载相关软件

### 部署Oracle客户端

### 部署ORACLE_FDW插件

### PSQL配置组件

```
[root@localhost oracle_fdw-ORACLE_FDW_2_1_0]# make
gcc -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -DLINUX_OOM_SCORE_ADJ=0 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -fPIC -shared -o oracle_fdw.so oracle_fdw.o oracle_utils.o oracle_gis.o -L/usr/lib64 -Wl,-z,relro   -Wl,--as-needed  -

/lib/../lib64/crt1.o /lib/../lib64/crti.o /opt/rh/devtoolset-9/root/usr/lib/gcc/x86_64-redhat-linux/9/crtbegin.o -L/opt/rh/devtoolset-9/root/usr/lib/gcc/x86_64-redhat-linux/9 -L/opt/rh/devtoolset-9/root/usr/lib/gcc/x86_64-redhat-linux/9/../../../../lib64 -L/lib/../lib64 -L/usr/lib/../lib64 -L/opt/rh/devtoolset-9/root/usr/lib/gcc/x86_64-redhat-linux/9/../../.. -lclntsh -lgcc --push-state --as-needed -lgcc_s --pop-state -lc -lgcc --push-state --as-needed -lgcc_s --pop-state /opt/rh/devtoolset-9/root/usr/lib/gcc/x86_64-redhat-linux/9/crtend.o /lib/../lib64/crtn.o
/opt/rh/devtoolset-9/root/usr/libexec/gcc/x86_64-redhat-linux/9/ld: cannot find -lclntsh
collect2: error: ld returned 1 exit status


[root@localhost oracle_fdw-ORACLE_FDW_2_1_0]# cd $ORACLE_HOME
[root@localhost instantclient]# ll |grep libclntsh
lrwxrwxrwx. 1 root root        17 Aug 29 14:25 libclntsh.so -> libclntsh.so.11.1
-rwxrwxr-x. 1 root root  53865194 Aug 24  2013 libclntsh.so.11.1

[root@localhost instantclient]# pwd
/usr/local/instantclient
[root@localhost instantclient]# ln -s libclntsh.so.11.1 libclntsh.so
[root@localhost instantclient]# ll
total 186444
-rwxrwxr-x. 1 root root     25420 Aug 24  2013 adrci
-rw-rw-r--. 1 root root       439 Aug 24  2013 BASIC_README
-rwxrwxr-x. 1 root root     47860 Aug 24  2013 genezi
-r-xr-xr-x. 1 root root       368 Aug 24  2013 glogin.sql
lrwxrwxrwx. 1 root root        17 Aug 29 14:25 libclntsh.so -> libclntsh.so.11.1
-rwxrwxr-x. 1 root root  53865194 Aug 24  2013 libclntsh.so.11.1
-r-xr-xr-x. 1 root root   7996693 Aug 24  2013 libnnz11.so
-rwxrwxr-x. 1 root root   1973074 Aug 24  2013 libocci.so.11.1
-rwxrwxr-x. 1 root root 118738042 Aug 24  2013 libociei.so
-r-xr-xr-x. 1 root root    164942 Aug 24  2013 libocijdbc11.so
-r-xr-xr-x. 1 root root   1502287 Aug 24  2013 libsqlplusic.so
-r-xr-xr-x. 1 root root   1469542 Aug 24  2013 libsqlplus.so
-r--r--r--. 1 root root   2091135 Aug 24  2013 ojdbc5.jar
-r--r--r--. 1 root root   2739616 Aug 24  2013 ojdbc6.jar
drwxrwxr-x. 4 root root        84 Aug 24  2013 sdk
-r-xr-xr-x. 1 root root      9320 Aug 24  2013 sqlplus
-rw-rw-r--. 1 root root       443 Aug 24  2013 SQLPLUS_README
-rwxrwxr-x. 1 root root    192365 Aug 24  2013 uidrvci
-rw-rw-r--. 1 root root     66779 Aug 24  2013 xstreams.jar
[root@localhost oracle_fdw-ORACLE_FDW_2_1_0]# make
gcc -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -DLINUX_OOM_SCORE_ADJ=0 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -fPIC -shared -o oracle_fdw.so oracle_fdw.o oracle_utils.o oracle_gis.o -L/usr/lib64 -Wl,-z,relro   -Wl,--as-needed  -L/usr/local/instantclient -L/usr/local/instantclient/bin -L/usr/local/instantclient/lib -lclntsh -L/usr/lib/oracle/12.2/client/lib -L/usr/lib/oracle/12.2/client64/lib -L/usr/lib/oracle/12.1/client/lib -L/usr/lib/oracle/12.1/client64/lib -L/usr/lib/oracle/11.2/client/lib -L/usr/lib/oracle/11.2/client64/lib -L/usr/lib/oracle/11.1/client/lib -L/usr/lib/oracle/11.1/client64/lib -L/usr/lib/oracle/10.2.0.5/client/lib -L/usr/lib/oracle/10.2.0.5/client64/lib -L/usr/lib/oracle/10.2.0.4/client/lib -L/usr/lib/oracle/10.2.0.4/client64/lib -L/usr/lib/oracle/10.2.0.3/client/lib -L/usr/lib/oracle/10.2.0.3/client64/lib 
[root@localhost oracle_fdw-ORACLE_FDW_2_1_0]# make install
/usr/bin/mkdir -p '/usr/lib64/pgsql'
/usr/bin/mkdir -p '/usr/share/pgsql/extension'
/usr/bin/mkdir -p '/usr/share/pgsql/extension'
/usr/bin/mkdir -p '/usr/share/doc/pgsql/extension'
/bin/sh /usr/lib64/pgsql/pgxs/src/makefiles/../../config/install-sh -c -m 755  oracle_fdw.so '/usr/lib64/pgsql/oracle_fdw.so'
/bin/sh /usr/lib64/pgsql/pgxs/src/makefiles/../../config/install-sh -c -m 644 ./oracle_fdw.control '/usr/share/pgsql/extension/'
/bin/sh /usr/lib64/pgsql/pgxs/src/makefiles/../../config/install-sh -c -m 644 ./oracle_fdw--1.1.sql ./oracle_fdw--1.0--1.1.sql  '/usr/share/pgsql/extension/'
/bin/sh /usr/lib64/pgsql/pgxs/src/makefiles/../../config/install-sh -c -m 644 ./README.oracle_fdw '/usr/share/doc/pgsql/extension/'
[root@localhost oracle_fdw-ORACLE_FDW_2_1_0]# ldd /usr/lib64/pgsql/oracle_fdw.so
    linux-vdso.so.1 =>  (0x00007ffe49144000)
    libclntsh.so.11.1 => /usr/local/instantclient/libclntsh.so.11.1 (0x00007ffa3787f000)
    libc.so.6 => /lib64/libc.so.6 (0x00007ffa374b1000)
    libnnz11.so => /usr/local/instantclient/libnnz11.so (0x00007ffa370e4000)
    libdl.so.2 => /lib64/libdl.so.2 (0x00007ffa36ee0000)
    libm.so.6 => /lib64/libm.so.6 (0x00007ffa36bde000)
    libpthread.so.0 => /lib64/libpthread.so.0 (0x00007ffa369c2000)
    libnsl.so.1 => /lib64/libnsl.so.1 (0x00007ffa367a8000)
    libaio.so.1 => /lib64/libaio.so.1 (0x00007ffa365a6000)
    /lib64/ld-linux-x86-64.so.2 (0x00007ffa3a1ee000)
[root@localhost oracle_fdw-ORACLE_FDW_2_1_0]# su - postgres
Last login: Sat Aug 29 14:31:27 EDT 2020 on pts/0
[postgres@localhost ~]$ psql
psql (12.2)
Type "help" for help.

postgres=# create extension oracle_fdw;
ERROR:  could not open extension control file "/usr/local/pg12.2/share/postgresql/extension/oracle_fdw.control": No such file or directory
postgres=# exit
postgres=#  create extension oracle_fdw;
ERROR:  could not load library "/usr/local/pg12.2/lib/postgresql/oracle_fdw.so": libclntsh.so.11.1: cannot open shared object file: No such file or directory
postgres=# exit
[postgres@localhost ~]$ pg_ctl restart
waiting for server to shut down.... done
server stopped
waiting for server to start....2020-09-16 11:20:11.455 EDT [45376] LOG:  starting PostgreSQL 12.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 9.3.1 20200408 (Red Hat 9.3.1-2), 64-bit
2020-09-16 11:20:11.457 EDT [45376] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2020-09-16 11:20:11.458 EDT [45376] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2020-09-16 11:20:11.488 EDT [45377] LOG:  database system was shut down at 2020-09-16 11:20:11 EDT
2020-09-16 11:20:11.491 EDT [45376] LOG:  database system is ready to accept connections
 done
server started
[postgres@localhost ~]$ psql
psql (12.2)
Type "help" for help.

postgres=#  create extension oracle_fdw;
CREATE EXTENSION
postgres=# \dx
                        List of installed extensions
    Name    | Version |   Schema   |              Description               
------------+---------+------------+----------------------------------------
 oracle_fdw | 1.1     | public     | foreign data wrapper for Oracle access
 plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)

postgres=# CREATE SERVER ora203 FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver '//192.168.1.203:1521/PROD');
CREATE SERVER
postgres=# create user scott_pg password 'scott';
CREATE ROLE
postgres=# GRANT USAGE ON FOREIGN SERVER ora203 TO scott_pg;  
GRANT
postgres=# CREATE USER MAPPING FOR scott_pg SERVER ora203 OPTIONS (user 'scott', password 'scott'); 
CREATE USER MAPPING
postgres=# exit
[postgres@localhost ~]$ psql -U scott_pg -d testdb
2020-09-16 11:25:31.968 EDT [45654] FATAL:  database "testdb" does not exist
psql: error: could not connect to server: FATAL:  database "testdb" does not exist
[postgres@localhost ~]$ psql -U scott_pg -d postgres
psql (12.2)
Type "help" for help.

postgres=> \d
Did not find any relations.
postgres=> CREATE FOREIGN TABLE ora_emp (
postgres(>  EMPNO      int,
postgres(>  ENAME      VARCHAR(10),
postgres(>  JOB        VARCHAR(9),
postgres(>  MGR        int,
postgres(>  HIREDATE   date,
postgres(>  SAL        float4,
postgres(>  COMM       float4,
postgres(>  DEPTNO     int
postgres(> ) SERVER ora203 OPTIONS (schema 'SCOTT', table 'EMP');  
CREATE FOREIGN TABLE
postgres=> \d
              List of relations
 Schema |  Name   |     Type      |  Owner   
--------+---------+---------------+----------
 public | ora_emp | foreign table | scott_pg
(1 row)

postgres=> select count(*) from ora_emp;
 count 
-------
    14
(1 row)
```



### 验证联通性

验证结果

```
postgres=> select * from ora_emp;
 empno | ename  |    job    | mgr  |  hiredate  | sal  | comm | deptno 
-------+--------+-----------+------+------------+------+------+--------
  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800 |      |     20
  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600 |  300 |     30
  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250 |  500 |     30
  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975 |      |     20
  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250 | 1400 |     30
  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850 |      |     30
  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450 |      |     10
  7788 | SCOTT  | ANALYST   | 7566 | 1987-04-19 | 3000 |      |     20
  7839 | KING   | PRESIDENT |      | 1981-11-17 | 5000 |      |     10
  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500 |    0 |     30
  7876 | ADAMS  | CLERK     | 7788 | 1987-05-23 | 1100 |      |     20
  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950 |      |     30
  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000 |      |     20
  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300 |      |     10
(14 rows)
```



### 参考文章

- [参考文章](http://www.postgres.cn/v2/news/viewone/1/337)