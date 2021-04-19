---
title:[PG]-事务进入读提交、可重复读、序列化模式的方法0
date:2020-11-09
---



​	把事务进入到READ COMMITTED、LEVEL REPEATABLE READ、SERIALIZABLE模式方法讲解。

### READ COMMITTED模式

```
postgres=# begin;
BEGIN
postgres=# set default_transaction_isolation='READ COMMITTED';
SET
postgres=# show transaction_isolation;
 transaction_isolation 
-----------------------
 repeatable read
(1 row)

postgres=# commit;
COMMIT
postgres=# show transaction_isolation;
 transaction_isolation 
-----------------------
 read committed
```



### REPEATABLE READ模式

```
postgres=# begin;
BEGIN
postgres=# set default_transaction_isolation='repeatable read';
SET
postgres=# commit;
COMMIT
postgres=# show transaction_isolation;
 transaction_isolation 
-----------------------
 repeatable read
(1 row)
```



### SERIALIZABLE模式

```
postgres=# begin;
BEGIN
postgres=# set transaction isolation level serializable;
SET
postgres=# show transaction_isolation;
 transaction_isolation 
-----------------------
 serializable
(1 row)
```