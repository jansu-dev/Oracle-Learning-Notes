---
title:[MySQL]--验证分区表建立的索引全部为本地(local)索引
date:2020-12-16
---



##### 测试MySQL删除分区表中分区对索引的影响

- [构建测试分区表](构建测试分区表)
- [新建索引观察变化](新建索引观察变化)
- [测试删分区对索引的影响](测试删分区对索引的影响)
- [结论](结论)
- [参考文章](参考文章)



### 构建测试分区表

- **数据数据库**

```
CREATE DATABASE  `jan` DEFAULT CHARACTER SET utf8;
```

- **创建分区表结构**

```
create table jan_test (
id int,
gname varchar(20),
mail_number int,
price varchar(20),
stock_number int,
create_time date,
primary key (id,create_time)
)ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT
partition by range columns (create_time)
(
partition p_20200101 values less than ('2020-11-01') engine=InnoDB,
partition p_20200201 values less than ('2020-12-01') engine=InnoDB,
partition p_20200301 values less than ('2021-01-01') engine=InnoDB
);
```

- **建立function随机字符串函数**

```
DROP FUNCTION IF EXISTS `rand_string`;
    DELIMITER // -- 替换语句默认的执行符号，将；替换成 //
    CREATE FUNCTION `rand_string` (n INT) RETURNS VARCHAR(255) CHARSET 'utf8'
    BEGIN 
        DECLARE char_str varchar(200) DEFAULT '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ';
        DECLARE return_str varchar(255) DEFAULT '';
        DECLARE i INT DEFAULT 0;
        WHILE i < n DO
            SET return_str = concat(return_str, substring(char_str, FLOOR(1 + RAND()*36), 1));
            SET i = i+1;
        END WHILE;
        RETURN return_str;
    END 
    //
    delimiter;
```

- **建立procedure插入一定的数据**

插入第一个分区数据的procedure；

```
DROP PROCEDURE IF EXISTS `batchInsertTestData_12`;

    DELIMITER //
    CREATE PROCEDURE `batchInsertTestData_12` (n INT) 
    BEGIN 
        DECLARE i INT DEFAULT 0;
        WHILE i < n DO
                    insert into jan_test(id,gname,mail_number,price,stock_number,create_time) 
                    values(i,rand_string(6),i,ROUND(RAND()*100),FLOOR(RAND()*100),now());
            SET i = i+1;
        END WHILE;
    END 
    //
    delimiter ;
```

插入第二个分区数据的procedure；

```
DROP PROCEDURE IF EXISTS `batchInsertTestData_11`;

    DELIMITER //
    CREATE PROCEDURE `batchInsertTestData_11` (n INT) 
    BEGIN 
        DECLARE i INT DEFAULT 0;
        WHILE i < n DO
                    insert into jan_test(id,gname,mail_number,price,stock_number,create_time) 
                    values(i+1000,rand_string(6),i,ROUND(RAND()*100),FLOOR(RAND()*100),date_sub(now(), interval 1 month));
            SET i = i+1;
        END WHILE;
    END 
    //

    delimiter ;
```

插入第3个分区数据的procedure；

```
DROP PROCEDURE IF EXISTS `batchInsertTestData_10`;

    DELIMITER //
    CREATE PROCEDURE `batchInsertTestData_10` (n INT) 
    BEGIN 
        DECLARE i INT DEFAULT 0;
        WHILE i < n DO
                    insert into jan_test(id,gname,mail_number,price,stock_number,create_time) 
                    values(i+2000,rand_string(6),i,ROUND(RAND()*100),FLOOR(RAND()*100),date_sub(now(), interval 2 month));
            SET i = i+1;
        END WHILE;
    END 
    //

    delimiter ;
```

- **调用procedure插入数据**

```
call batchInsertTestData_12(1000);
    call batchInsertTestData_11(1000);
    call batchInsertTestData_10(1000);
```

- **测试分区数据是否插入**

```
mysql> select count(*) from jan_test where create_time > date '2020-10-01'and create_time < date '2020-12-01';
++
| count(*) |
++
|     1000 |
++
1 row in set (0.00 sec)
```

- **建立分区表后的空表**

```
[root@JanDB03 jan]# ll
total 304
-rw-r-----. 1 mysql mysql    61 Dec 15 23:08 db.opt
-rw-r-----. 1 mysql mysql  8754 Dec 15 23:10 jan_test.frm
-rw-r-----. 1 mysql mysql 98304 Dec 15 23:10 jan_test#P#p_20200101.ibd
-rw-r-----. 1 mysql mysql 98304 Dec 15 23:10 jan_test#P#p_20200201.ibd
-rw-r-----. 1 mysql mysql 98304 Dec 15 23:10 jan_test#P#p_20200301.ibd
```

- **建立数据文件截图插入一定的数据**

可以看到随着数据的插入，各分区的数据文件（jan_test#P#p_20200101.ibd等等）开始膨胀。

```
[root@JanDB03 jan]# ll
total 448
-rw-r-----. 1 mysql mysql     61 Dec 15 23:08 db.opt
-rw-r-----. 1 mysql mysql   8754 Dec 15 23:10 jan_test.frm
-rw-r-----. 1 mysql mysql 147456 Dec 16 01:36 jan_test#P#p_20200101.ibd
-rw-r-----. 1 mysql mysql 147456 Dec 16 01:36 jan_test#P#p_20200201.ibd
-rw-r-----. 1 mysql mysql 147456 Dec 16 01:35 jan_test#P#p_20200301.ibd
```



### 新建索引观察变化

- **建立索引前**

```
mysql> show index from jan_test\G
*************************** 1. row ***************************
        Table: jan_test
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 3000
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
*************************** 2. row ***************************
        Table: jan_test
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 2
  Column_name: create_time
    Collation: A
  Cardinality: 3000
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
2 rows in set (0.00 sec)
```

- **非主键字段建立索引**

```
mysql> create index ind_jan_mail_number on jan_test(mail_number);
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0



mysql> show index from jan_test\G                              
*************************** 1. row ***************************
        Table: jan_test
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 3000
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
*************************** 2. row ***************************
        Table: jan_test
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 2
  Column_name: create_time
    Collation: A
  Cardinality: 3000
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
*************************** 3. row ***************************
        Table: jan_test
   Non_unique: 1
     Key_name: ind_jan_mail_number
 Seq_in_index: 1
  Column_name: mail_number
    Collation: A
  Cardinality: 3000
     Sub_part: NULL
       Packed: NULL
         Null: YES
   Index_type: BTREE
      Comment: 
Index_comment: 
3 rows in set (0.00 sec)
```

- **建立索引后数据文件截图**

可以看到建立ind_jan_mail_number索引后，数据文件大小由147456膨胀至212992，说明在MySQL的分区表中索引和分区数据是存在同一文件之中的。

```
[root@JanDB03 jan]# ll
total 640
-rw-r-----. 1 mysql mysql     61 Dec 15 23:08 db.opt
-rw-r-----. 1 mysql mysql   8754 Dec 16 01:42 jan_test.frm
-rw-r-----. 1 mysql mysql 212992 Dec 16 01:42 jan_test#P#p_20200101.ibd
-rw-r-----. 1 mysql mysql 212992 Dec 16 01:42 jan_test#P#p_20200201.ibd
-rw-r-----. 1 mysql mysql 212992 Dec 16 01:42 jan_test#P#p_20200301.ibd
```



### 测试删分区对索引的影响

- **SQL执行计划查询**

```
mysql> explain select * from jan_test where mail_number=10\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: jan_test
   partitions: p_20200101,p_20200201,p_20200301
         type: ref
possible_keys: ind_jan_mail_number
          key: ind_jan_mail_number
      key_len: 5
          ref: const
         rows: 3
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

- **删除索引测试执行计划**

```
mysql> alter table jan_test drop partition p_20200201;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from jan_test where mail_number=10\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: jan_test
   partitions: p_20200101,p_20200301
         type: ref
possible_keys: ind_jan_mail_number
          key: ind_jan_mail_number
      key_len: 5
          ref: const
         rows: 2
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

- **删除索引文件大小截图**

```
[root@JanDB03 jan]# ll

total 432
-rw-r-----. 1 mysql mysql     61 Dec 15 23:08 db.opt
-rw-r-----. 1 mysql mysql   8754 Dec 16 01:47 jan_test.frm
-rw-r-----. 1 mysql mysql 212992 Dec 16 01:42 jan_test#P#p_20200101.ibd
-rw-r-----. 1 mysql mysql 212992 Dec 16 01:42 jan_test#P#p_20200301.ibd
```



### 结论

最终结论，MySQL中没有Oracle全局索引的概念，所有的分区索引均存在对应的每个分区数据文件中；
在删除索引时，可以直接使用drop paratition命令删除分区，该操作不会对索引的使用造成影响。



### 参考文章

- [pursuer.chen的文章](https://www.cnblogs.com/chenmh/p/5761995.html)