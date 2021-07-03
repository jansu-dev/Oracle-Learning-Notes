---
title:OGG常用命令
date:2020-08-17
---



### OGG常用命令整理：

ogg常用命令整理
stats --查看进程抽取数据情况，用于检测数据丢失问题。
info all：查看所有增量抽取进程以及增量推送进程
info all,task：用于查看全量抽取进程
stop *!：强行停止进程
kill：杀掉无法停止的进程
view params \*：查看进程所需的配置信息
edit params \*：编辑进程所需的配置信息
info exittrail \*：查看进程创建时的信息
view report \*：查看进程运行时log信息
info credentialstore： 查看配置用户信息，用于用户名、存储密码，为了安全。注意：CredentialSore是 ogg 12c 新特性
add credentialstore：添加配置用户信息文件。
delete credentialstore：删除用户配置文件。
alter credentialstore add user xx* password xx* alias xx* ：分别为 用户名 、密码 、别名
alter credentialstore delete user xx* : 删除一个用户信息
dblogin userid xx*,password xx\*：数据库登陆，明文
dblogin useridalias xx\*：数据库登陆，使用用户的别名alias登陆
alter \* ,extseqno 0, extrba 0：extseqno 设置读取文件的位置，extrba 设置读取文件内容位置
add extract e*, tranlog, begin now：创建增量进程—>配置相应的配置文件
add exttrail ./dirdat/xx*,extract e_gessdb,megabytes 100：配置进程
add extract p*,exttrailsource ./dirdat/xx*：创建推送进程—>配置相应的配置文件
add extract i*, sourceistable：创建全量进程—>配置相应的配置文件

```
--前滚重新生成一个新的队列文件

alter extract exxxx etrollover
附：
--从指定时间重新抓取（重新抓取数据前提：归档文件没有删除）

alter extract xxxxx, tranlog, begin 2018-12-31 08:00

--重置抽取进程，本地文件序列号从0开始生成。

alter extract exxxx,extseqno 0,extrba 0

--重置读取进程，重新从0号trial文件开始读取。

alter replicat rxxxx,extseqno 0,extrba 0
```

alter extract AAA, at scn YOUR_BREAK_POINT_SCN;从某scn时间点开始投递。



当主库删除备库不存在的数据库，会导致备库OGG进程挂掉，可使用skiptransaction解决。

```
start rep2 skiptransaction
```

丢失update的更新操作是针对主键的更新，此时replicat会尝试插入一条记录而非忽略该update。 注意插入的记录可能不是完整的行，如上例中的T2 为NULL .
若要求完整的行记录则要求EXTRACT使用PKUPDATE选项。 需要加入的选项是FETCHOPTIONS FETCHPKUPDATECOLS 将以上选项加入到EXTRACT参数文件中，并重启EXTRACT



### 参考文章

- [参考文章1-handle collsion](https://www.cnblogs.com/macleanoracle/archive/2013/03/19/2968339.html)

- 当表结构不一致时，使用colmap进行表结构映射。
  - [参考文章2-colmap](http://blog.itpub.net/31485142/viewspace-2151879/)
  - [参考文章3-colmap映射实验](http://www.360doc.com/content/19/0509/17/2245786_834614490.shtml)