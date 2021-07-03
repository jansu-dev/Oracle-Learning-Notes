---
title:[MySQL]--MySQL5.7具备视图谓词自动下推功能
date:2020-08-10
---



##### MySQL5.7视图谓词自动下推功能的实验验证。

### 实验环境

```
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-1062.el7.x86_64 #1 SMP Wed Aug 7 18:08:02 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

mysql> select version();
+------------+
| version()  |
+------------+
| 5.7.30-log |
+------------+
```



### 实验准备

学生表 Student

```
create table Student(SId varchar(10) primary key,Sname varchar(10),Sage datetime,Ssex varchar(10));
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('04' , '赵雷' , '1999-09-09' , '男');
insert into Student values('02' , '琳琳' , '1990-12-21' , '女');
insert into Student values('03' , '孙风' , '1990-05-20' , '男');
```

科目表 Course

```
create table Course(CId varchar(10) primary key,Cname nvarchar(10));
insert into Course values('01' , '语文');
insert into Course values('02' , '数学');
insert into Course values('03' , '英语');
```

成绩表 SC

```
create table SC(SId varchar(10),CId varchar(10),score decimal(18,1));
insert into SC values('01' , '01' , 80);
insert into SC values('01' , '02' , 90);
insert into SC values('01' , '03' , 99);
insert into SC values('02' , '01' , 70);
insert into SC values('02' , '02' , 60);
insert into SC values('02' , '03' , 80);
insert into SC values('03' , '01' , 80);
insert into SC values('03' , '02' , 80);
```

创建索引

```
create index ind_name on Student(Sname);
```

选课视图 V_ChooseClass

```
create view V_ChooseClass as select s.SId, s.Sname, count(c.CId) as CId from Student s left join SC c on s.SId=c.SId group by s.SId, s.Sname;
```

查询"赵雷"同学的学生编号、学生姓名、选课总数

```
select s.SId, s.Sname, count(c.CId) as CId from Student s left join SC c on s.SId=c.SId where s.Sname='赵雷' group by c.SId;

select SId, Sname, CId from V_ChooseClass where Sname='赵雷';
```



### 实验结果

```
mysql> explain select s.SId, s.Sname, count(c.CId) from Student s left join SC c on s.SId=c.SId where s.Sname='赵雷' group by s.SId, s.Sname;
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | s     | NULL       | ref  | ind_name      | ind_name | 33      | const |    2 |   100.00 | Using index; Using temporary; Using filesort       |
|  1 | SIMPLE      | c     | NULL       | ALL  | NULL          | NULL     | NULL    | NULL  |    8 |   100.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> explain select SId, Sname, CId from V_ChooseClass where Sname='赵雷';
+----+-------------+------------+------------+-------+---------------+-------------+---------+-------+------+----------+----------------------------------------------------+
| id | select_type | table      | partitions | type  | possible_keys | key         | key_len | ref   | rows | filtered | Extra                                              |
+----+-------------+------------+------------+-------+---------------+-------------+---------+-------+------+----------+----------------------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 33      | const |    3 |   100.00 | NULL                                               |
|  2 | DERIVED     | s          | NULL       | index | ind_name      | ind_name    | 33      | NULL  |    4 |   100.00 | Using index; Using temporary; Using filesort       |
|  2 | DERIVED     | c          | NULL       | ALL   | NULL          | NULL        | NULL    | NULL  |    8 |   100.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+------------+------------+-------+---------------+-------------+---------+-------+------+----------+----------------------------------------------------+
3 rows in set, 1 warning (0.00 sec)
```

### 实验结论

MySQL5.7版本具备谓词自动下推功能。