# Oracle-Learning-Notes
That the repository was build is aim to log process of my Oracle Learning.And,there are some video which is made by myself for the page reader.

历史文章会慢慢由个人网站搬迁过来,  
因为网站运行在个人电脑上虚拟机中,  
因工作原因，时而可能访问不到，如果时B站观看分享的伙伴  
可以通过邮箱联系我coresu@icloud.com  
BLOG网址：http://www.jannest.com/  


# 目录

> - [LeetCode](#LeetCode)
> - [MongoDB](#MongoDB)
> - [MySQL](#MySQL)
> - [Oracle](#Oracle)
> - [SQL Sever](#SQLSever)
> - [基础知识](#基础知识)
> - [破解资源](#破解资源)









## **LeetCode**

- [singleNumber解法](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/LeetCode/singleNumber%E8%A7%A3%E6%B3%95.md)



## MongoDB

- [Mongo导入navicat导出的json数据失败.md](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/MongoDB/Mongo%E5%AF%BC%E5%85%A5navicat%E5%AF%BC%E5%87%BA%E7%9A%84json%E6%95%B0%E6%8D%AE%E5%A4%B1%E8%B4%A5.md)



## MySQL

- [用户管理](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/MySQL/%E7%94%A8%E6%88%B7%E7%AE%A1%E7%90%86.md)



## Oracle

- #### Oracle-DG

  - ##### DG-DataGuard

    - [为什么用DG](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-DataGuard/%E4%B8%BA%E4%BB%80%E4%B9%88%E7%94%A8DG.md)
    - [什么是DG](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-DataGuard/%E4%BB%80%E4%B9%88%E6%98%AFDG.md)

  - ##### DG-基本概念

    - [DG三种模式](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5/DG%E4%B8%89%E7%A7%8D%E6%A8%A1%E5%BC%8F.md)
    - [DG基本概念](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5/DG%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5.md)

  - ##### DG-安装与部署

    - [Oracle11g部署DG](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-%E5%AE%89%E8%A3%85%E4%B8%8E%E9%83%A8%E7%BD%B2/Oracle11g%E9%83%A8%E7%BD%B2DG.md)

  - ##### DG-常见问题

    - [switchover-status出现sessions-active](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/switchover_status%E5%87%BA%E7%8E%B0sessions%20active.md)
    - [防火墙与SELINUX导致DG主备库失连接](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-DG/DG-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/%E9%98%B2%E7%81%AB%E5%A2%99%E4%B8%8ESELINUX%E5%AF%BC%E8%87%B4DG%E4%B8%BB%E5%A4%87%E5%BA%93%E5%A4%B1%E8%BF%9E%E6%8E%A5.md)

- #### Oracle-OGG

  - ##### OGG-基本概念

    - [OGG文档学习](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-基本概念/OGG文档学习.md)
    - [OGG进程配置解析](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-基本概念/OGG进程配置解析.md)

  - ##### OGG-常见问题

    - [【OGG】OCIErrorORA-02291违反完整约束条件](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-常见问题/[OGG]OCIErrorORA-02291违反完整约束条件.md)
    - [【OGG】寻找造成OGG延时的原因](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-常见问题/[OGG]寻找造成OGG延时的原因.md)
    - [【OGG】导致单向联通失败的几点原因](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-常见问题/[OGG]导致单向联通失败的几点原因.md)
    - [【OGG】解决运行@ddl_setup.sql时报错及ORA-04098- 触发器 'SYS.GGS_DDL_TRIGGER_BEFORE' 无效](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/%E3%80%90OGG%E3%80%91%E8%A7%A3%E5%86%B3%E8%BF%90%E8%A1%8C%40ddl_setup.sql%E6%97%B6%E6%8A%A5%E9%94%99%E5%8F%8AORA-04098-%20%E8%A7%A6%E5%8F%91%E5%99%A8%20'SYS.GGS_DDL_TRIGGER_BEFORE'%20%E6%97%A0%E6%95%88.md)
    - [【OGG】解读DYNAMICRESOLUTION参数](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-OGG/OGG-常见问题/[OGG]解读DYNAMICRESOLUTION参数.md)

- #### Oracle-RAC/RAC常见问题

  - [【RAC】NFS权限问题root出现Permission denied](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-RAC/RAC-常见问题/[RAC]NFS权限问题root出现Permission denied.md)
  - [【RAC】RAC安装过程提示not a shared subnet错误](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-RAC/RAC-常见问题/[RAC]RAC安装过程提示not a shared subnet错误.md)
  - [【RAC】历史及简介](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-RAC/RAC-常见问题/[RAC]历史及简介.md)
  - [【RAC】常用命令总结](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-RAC/RAC-常见问题/[RAC]常用命令总结.md)
  - [【RAC】节点1的trace日志出现minact-scn致使数据库处于mounted态无法开库](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-RAC/RAC-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/%E3%80%90RAC%E3%80%91%E8%8A%82%E7%82%B91%E7%9A%84trace%E6%97%A5%E5%BF%97%E5%87%BA%E7%8E%B0minact-scn%E8%87%B4%E4%BD%BF%E6%95%B0%E6%8D%AE%E5%BA%93%E5%A4%84%E4%BA%8Emounted%E6%80%81%E6%97%A0%E6%B3%95%E5%BC%80%E5%BA%93.md)

- #### Oracle-SQL调优

  - [应用发版新SQL缺索引导致全表扫](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-SQL调优/应用发版新SQL缺索引导致全表扫.md)

- #### Oracle-体系

  - [【体系】--Insert语句在Oracle实例中的的运作流程](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-体系/[体系]--Insert语句在Oracle实例中的的运作流程.md)
  - [【体系】--Oracle体系](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-体系/[体系]--Oracle体系.md)
  - [【体系】--select语句在Oracle实例中的运作过程](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-体系/[体系]--select语句在Oracle实例中的运作过程.md)
  - [【体系】--update语句在Oracle实例中的运作过程.md](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-体系/[体系]--update语句在Oracle实例中的运作过程.md)

- #### Oracle-常用SQL

  - [Kill正在执行的存储过程](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/Kill正在执行的存储过程.md)
  - [Linux查看IO负载处理能力](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/Linux查看IO负载处理能力.md)
  - [PLSQL抓出慢sql方法](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/PLSQL抓出慢sql方法.md)
  - [Procedure批量删除数据](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/Procedure批量删除数据.md)
  - [dbms_scheduler定时任务使用](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/dbms_scheduler定时任务使用.md)
  - [依据伪列建立分区表及子分区](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/依据伪列建立分区表及子分区.md)
  - [取分区表high-value中Long型日期](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常用SQL/取分区表high-value中Long型日期.md)

- #### Oracle-常见问题

  - [ASM的connect的和mounted的区别](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/ASM的connect的和mounted的区别.md)
  - [Oracle materialize view无法自动刷新问题](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Oracle materialize view无法自动刷新问题.md)
  - [Oracle--DBCA起不来的防火墙原因](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Oracle--DBCA起不来的防火墙原因.md)
  - [Oracle备份恢复之recover database的四条语句区别](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Oracle备份恢复之recover database的四条语句区别.md)
  - [Oracle数据库在truncate不能flashback table的原因](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Oracle数据库在truncate不能flashback table的原因.md)
  - [Oracle查询表空间使用情况以及其他查询](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Oracle查询表空间使用情况以及其他查询.md)
  - [Oracle正常安装sqlplus命令无法登录](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Oracle正常安装sqlplus命令无法登录.md)
  - [Shell子进程修改父进程环境变量值](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Shell子进程修改父进程环境变量值.md)
  - [Shell脚本提示No-such-file-or-directory解决方法](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/Shell脚本提示No-such-file-or-directory解决方法.md)
  - [recover-database-using-backup-control-file报system01.dbf文件修复](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/recover%20database%20using%20backup%20control%20file%E6%8A%A5system01.dbf%E6%96%87%E4%BB%B6%E4%BF%AE%E5%A4%8D.md)
  - [如何查看日志切换的时间](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/如何查看日志切换的时间.md)
  - [当数据库出现OC4J-Configuration-issue](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/当数据库出现OC4J-Configuration-issue.md)
  - [恢复数据库后以read-only打开提示recovery的原因](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/恢复数据库后以read-only打开提示recovery的原因.md)
  - [数据泵在调优过程中的作用](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/数据泵在调优过程中的作用.md)
  - [索引段坏块问题解决](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/索引段坏块问题解决.md)
  - [诊断Oracle-Redo-Log引发的性能问题](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/诊断Oracle-Redo-Log引发的性能问题.md)
  - [重建索引未考虑空间引发的问题](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/重建索引未考虑空间引发的问题.md)
  - [验证表空间是否自动回收空间](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-常见问题/验证表空间是否自动回收空间.md)

- #### Oracle-性能调优

  - ##### 20000000insert数据

    - [Archivelog-Logging-Append-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/Archivelog-Logging-Append-NoPartition.md)
    - [Archivelog-Logging-NoAppend-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/Archivelog-Logging-NoAppend-NoPartition.md)
    - [Archivelog-NoLogging-Append-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/Archivelog-NoLogging-Append-NoPartition.md)
    - [Archivelog-NoLogging-NoAppend-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/Archivelog-NoLogging-NoAppend-NoPartition.md)
    - [NoArchivelog-Logging-Append-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/NoArchivelog-Logging-Append-NoPartition.md)
    - [NoArchivelog-Logging-NoAppend-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/NoArchivelog-Logging-NoAppend-NoPartition.md)
    - [NoArchivelog-NoLogging-Append-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/NoArchivelog-NoLogging-Append-NoPartition.md)
    - [NoArchivelog-NoLogging-NoAppend-NoPartition](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/20000000insert数据/NoArchivelog-NoLogging-NoAppend-NoPartition.md)

  - [Hint方法指定SQL执行计划](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/Hint方法指定SQL执行计划.md)

  - [【调优】--如何优化批量insert数据](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/[调优]--如何优化批量insert数据.md)

  - [【调优】--解析Direct Path Read](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/[调优]--解析Direct Path Read.md)

  - [【调优】--解读10046事件](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/[调优]--解读10046事件.md)

  - [【调优】--解读10053事件](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/[调优]--解读10053事件.md)

  - [【调优】--解读自动维护任务](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-性能调优/[调优]--解读自动维护任务.md)

- #### Oracle-错误编码

  - [【错误】--ORA-01555--快照太旧与AUM介绍](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-错误编码/[错误]--ORA-01555--快照太旧与AUM介绍.md)
  - [【错误】--ORA-27101--shared-memory-realm不存在](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-错误编码/[错误]--ORA-27101--shared-memory-realm不存在.md)
  - [【错误】--ORA-29345--跨平台传输字符集问题解决](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-错误编码/[错误]--ORA-29345--跨平台传输字符集问题解决.md)
  - [【错误】--ORA-30036--无法按8扩展段](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/Oracle/Oracle-错误编码/[错误]--ORA-30036--无法按8扩展段.md)



## SQL sever

- ##### AlwaysOn高可用集群部署

  - [AlwaysOn高可用集群部署](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/SQL Server/AlwaysOn高可用集群部署/AlwaysOn高可用集群部署.pdf)

- [SQLServer使用LinkedServer连接Oracle(伪dblink)](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/SQL Server/SQLServer使用LinkedServer连接Oracle(伪dblink).md)

- [收缩日志空间](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/SQL Server/收缩日志空间.md)


## 基础知识

- [上下文context](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/基础知识/上下文context.md)
- [对裸设备的认识](https://github.com/jansu-dev/Oracle-Learning-Notes/blob/master/基础知识/对裸设备的认识.md)



## 第一部分 体系架构


> [**单实例ASM静默安装**](http://www.jannest.com/article/138)

> [**RAC装机规范文档**](http://www.jannest.com/article/146)

* **第一章：实例与数据库**

> [**1.1 Oracle 基础架构及应用环境**]()

> [**1.2 SGA的基本组件 【待补充】**]()

> [**1.3 Oracle的进程【待补充】**]()

> [**1.4 PGA的基本组件【待补充】**]()

> [**1.5连接方式【待补充】**]()


* **第二章：实例管理及数据库的启动/关闭**

2.1 参数文件【待补充】
    
2.2 数据库启动与关闭【待补充】

2.3 自动诊断信息库ADR【待补充】

2.4 口令文件【待补充】


* **第三章：控制文件**

3.1 功能和特点【待补充】

3.2 实时更新机制【待补充】

3.3 多元化【待补充】

3.4 备份与重建【待补充】

3.5 恢复与重建【待补充】



* **第四章：redo 日志**

4.1作用和特征【待补充】

4.2日志组及切换【待补充】

4.3 添加日志组和成员【待补充】

4.4 v$log视图的状态【待补充】

4.5日志恢复【待补充】

4.6 使日志恢复到原来的配置【待补充】


* **第五章：归档日志**

5.1 归档和非归档的区别【待补充】

5.2 设置归档模式【待补充】

5.3 路径及命名方法【待补充】

5.4 归档进程和手动切换【待补充】


* **第六章：日志挖掘**

6.1 作用【待补充】

6.2 方法【待补充】


* **第七章：管理undo**

7.1 Undo作用【待补充】

7.2 Undo的参数【待补充】

7.3 Undo空间重用机制【待补充】

7.4关于AUM【待补充】

7.5 undo 信息的查询【待补充】

* **第八章：检查点（checkpoint)**

8.1什么是checkpoint【待补充】


8.2检查点分类【待补充】

* **第九章：实例恢复机制**
9.1什么是实例恢复【待补充】

9.2增量检查点发挥的作用【待补充】

* **第十章：手工创建数据库**

## 第二部分 存储架构

* **第十一章：数据字典**

11.1什么是数据字典【待补充】

11.2数据字典内容【待补充】

11.3数据字典组成【待补充】

11.4查询静态和动态视图【待补充】


* **第十二章：逻辑存储架构**

12.1 TABLESPACE(表空间）【待补充】

12.2 SEGMENT(段）【待补充】

12.3 EXTENT（区）【待补充】

12.4 BLOCK(数据块）【待补充】

12.5 临时表空间【待补充】

12.6 如何调整表空间的尺寸【待补充】


* **第十三章：表的类型**

13.1表的类型【待补充】

13.2表分区及其种类【待补充】

13.4 表的联机重定义【待补充】

13.5 索引组织表【待补充】

13.6 簇表(cluster table)【待补充】

13.8 临时表 【待补充】

13.9 只读表【待补充】

13.10 压缩表【待补充】

## 第三部分 网络、审计、字符集

* **第十四章：Oracle网络**

14.1 Oracle Net是什么【待补充】

14.2 配置文件【待补充】

14.3 轻松连接方式（ezconnect)【待补充】

14.4 动态注册【待补充】

14.5 静态注册【待补充】

14.6 客户端配置文件tnsnames.ora【待补充】

14.7 理解监听器【待补充】

14.8 共享连接配置【待补充】

* **第十五章：数据库审计audit**

15.1 功能和类别：【待补充】

15.2 启用审计（默认不启用）【待补充】

15.3 标准数据库审计的三个级别【待补充】

15.4 基于值的审计。 【待补充】

15.5 精细审计Fine Grained Auditing (FGA)。【待补充】

* **第十六章：全球化特性与字符集**

16.1 全球化特性内容【待补充】

16.2 字符集概念【待补充】

16.3字符集及分类【待补充】
 
16.4 Unicode字符集【待补充】

16.5 NLS参数设定【待补充】

## 第四部分 数据仓库管理

* **第十七章：数据移动**

17.1 概念【待补充】

17.2 Directory(目录)【待补充】

17.3 sql*loader【待补充】

17.4 外部表示例【待补充】

* **第十八章：逻辑备份（导出）与恢复（导入）**

18.1 传统的导入导出exp/imp：【待补充】

18.2 导入导出示例【待补充】

18.3 数据泵技术【待补充】

18.4 数据泵示例【待补充】

18.5 数据泵直传示例【待补充】

* **第十九章：物化视图**

19.1 产生和作用【待补充】

19.2 物化视图基本功能【待补充】

19.3 物化视图的操作【待补充】

> [**19.4 Database link**](http://www.jannest.com/article/163)

19.5 物化视图示例【待补充】

## 第五部分 集群部署与管理

* **20.1 RAC基本概念**

* **20.2 RAC的作用**

## 第六部分 中间件与容灾方案
