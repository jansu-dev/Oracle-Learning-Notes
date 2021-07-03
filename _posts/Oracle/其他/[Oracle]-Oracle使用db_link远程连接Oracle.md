---
title:[Oracle]-Oracle使用db_link远程连接Oracle
date:2020-12-09
---



Oracle使用db_link连接Oracle

- [DB_LINK概念](DB_LINK概念)
- [DB_LINK使用](DB_LINK使用)
- [DB_LINK相关视图](DB_LINK相关视图)
- [参考文章](参考文章)



### DB_LINK概念

- **基础概念**

database link是定义一个数据库到另一个数据库的路径的对象，database link允许你查询远程表及执行远程的存储过程；
database link使用Oracle Net做单向连接，在分布式环境里，database link用来跨库访问数据非常有效；
可以像访问本地数据库一样访问远程数据库表中的数据。

可由用户创建属于自己schema下的database link对象，该db_link仅可以被该schema调用；
也可由SYS用户或拥有DBA角色的用户（否则需要授权）创建public database link，该db_link可以被所有schema调用。

- **模式分类**

Privte： 用户级
Public ：数据库级
Globle ：网络级

- **操作语法**

```
CREATE [SHARED][PUBLIC] database link link_name
[CONNECT TO [user][current_user] IDENTIFIED BY password]
      [AUTHENTICATED BY user IDENTIFIED BY password]
     [USING 'connect_string']
```

[PUBLIC] 选项表示这个database link所有用户都可以使用；
[USING ‘网络连接符’] 这里描述的tnsnames里的描述；



### DB_LINK使用

创建DB_LINK

```
CREATE database link db68
CONNECT TO cp_tms IDENTIFIED BY "password"
USING '(DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.2.196.68)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = tmsdb)
    )
  )';
```

删除db_link

```
drop database link db68;
```

执行存储过程

```
select function_name @db_link_name(inpar1,inparm2...) into var from dual;
```



### DB_LINK相关视图

ALL_DB_LINKS ： describes the database links accessible to the current user.

DBA_DB_LINKS : describes all database links in the database.

USER_DB_LINKS : describes the database links owned by the current user. This view does not display the OWNER column.

DBA_DB_LINK_SOURCES :

DBA_EXTERNAL_SCN_ACTIVITY ：DBA_EXTERNAL_SCN_ACTIVITY与DBA_DB_LINK_SOURCES和DBA_DB_LINKS视图一起使用，以确定高SCN活动的来源



### 参考文章

- [参考文章1-Database Reference--DBA_DB_LINK_SOURCES](https://docs.oracle.com/en/database/oracle/-oracle-database/18/refrn/DBA_DB_LINK_SOURCES.html#GUID-0E6F0B9A-F816-4791-8DCB-00623DDD7456)

- [参考文章2-百度百科--dblink](https://baike.baidu.com/item/dblink/15079163?fr=aladdin)