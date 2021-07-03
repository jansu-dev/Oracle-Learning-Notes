---
title:[SQL调优]--Oracle10g优化器因正确的统计信息选择高消耗的执行计划
date:2020-09-03
---



​     [SQL调优]--Oracle10g优化器因正确的统计信息选择高消耗的执行计划



### 案例背景

AWR报告:

##### **基本信息**

节点一：
![image.png](http://cdn.lifemini.cn/dbblog/20200903/f458a07d373f4a7c98f0e2e31094149a.png)
节点二：
![image.png](http://cdn.lifemini.cn/dbblog/20200903/2bc7478576604376b59454ca3abb56ec.png)

**Load Profile**
节点一从AWR报告看，逻辑读达到1.3G/s的逻辑读，节点二甚至更高。
节点一：
![image.png](http://cdn.lifemini.cn/dbblog/20200903/5594af99d2f2402488b2959a07fd99b2.png)
节点二：
![image.png](http://cdn.lifemini.cn/dbblog/20200903/3b82cabe246c425cbc8726ea1ed9f123.png)

**Top 5 Timed Events**
节点一基本反应在I/O方面存在压力，db file scattered read和db file sequential read的平均等待分别为30ms和37ms，与正常情况平均等待保持在10ms以内为标准相比，节点一的压力很大。

节点一：
![image.png](http://cdn.lifemini.cn/dbblog/20200903/322990eca9a44e5eb0413128eff45fee.png)

节点二：
![image.png](http://cdn.lifemini.cn/dbblog/20200903/4a3ed0fa9633452e8c49c1e592d14935.png)



### 问题原因:

在SQL statistics中，最终定位到问题SQL。
节点一：
![image.png](http://cdn.lifemini.cn/dbblog/20200907/4e280f3226a44a70ad640447d7cf7f80.png)

节点二：
![image.png](http://cdn.lifemini.cn/dbblog/20200907/5ab52f15efb545a686cf59613f9cf559.png)

通过与开发沟通，发现实际上这是一个分页语句；
会先count总页数，再依据传过来的参数查询分页所需信息,
再次本次选择非count语句展示并优化。
**企业信息已处理**

```
select *
    from (select XXX_customer0_.customer_no        as customer1_,
                 XXX_customer0_.customer_name      as customer2_,
                 XXX_customer0_.exist_position     as exist_po3_,
                 XXX_customer0_.belongs_solicitor  as belongs_4_,
                 XXX_customer0_.depart_library     as depart_l5_,
                 XXX_customer0_.allot_apply_flag   as allot_ap6_,
                 XXX_customer0_.allot_user         as allot_user,
                 ......                           
                 ......
                 XXX_customer0_.last_visit_sample  as last_vi57_,
                 XXX_customer0_.last_visit_type    as last_vi58_,
                 XXX_customer0_.contractValDate    as contrac59_,
                 XXX_customer0_.isContractSample   as isContr60_,
                 XXX_customer0_.isContractMoney    as isContr61_,
                 XXX_customer0_.isContractApply    as isContr62_,
                 XXX_customer0_.contractNo         as contractNo
            from TBL_CUSTOMER XXX_customer0_
           where (XXX_customer0_.customer_no <> "id12312s2qx3_n1")
             and (exists
                  (select distinct XXX_contact1_.customer_no
                     from TBL_JAN_CONTACT XXX_contact1_
                    where (XXX_contact1_.customer_no = XXX_customer0_.customer_no)
                      and (Lower(XXX_contact1_.ALLCONTECT) like "%1364428%"))))
   where rownum <= :1
```

定位到问题SQL以后，发现更加难办！
在谓词位置分别使用了“不等于”和like左右具有百分号。

查看具体字段存储内容，发现这是一张与客户信息有关的表；
如图所示，一个客户可能有多种联系方式，有座机电话、手机号码、E-mail等；
可开发时候采用的方式却偏偏是，将所有的联系方式拼接到一个字段中，最终like模糊搜索用户信息。

![image.png](http://cdn.lifemini.cn/dbblog/20200907/975e6ea6d69d49bc8c120ea0e6bb1506.png)

该条SQL的执行计划却又偏偏是全表扫描；
![image.png](http://cdn.lifemini.cn/dbblog/20200911/a626d4d19fff476d854dd82a6725e157.png)

该条SQL的实际的执行时间为24秒；
![3.png](http://cdn.lifemini.cn/dbblog/20200907/bce2cdd561074177bebda9ab339126f6.png)

简单的补建索引之后，发现作用并不大。

![image.png](http://cdn.lifemini.cn/dbblog/20200911/8bc726a35acc4aceae5d380ce8254a40.png)

```sql
create index TBL_JAN_CUSTOMER_KEY on TBL_JAN_CUSTOMER(customer_no) online; 

create index TBL_JANCUSTOMER_COMBINE on TBL_JAN_CUSTOMER(customer_no,Lower(TBL_JAN_CONTACT.ALLCONTECT)) online;
```

![6.png](http://cdn.lifemini.cn/dbblog/20200907/1dcd8a3d571f4a2fbf71845d625435ca.png)

虽然使用并行可以经运行时间缩小到7s，但是作为应用端发送来的SQL，不适合随意使用并行。
当应用的并发请求增多时，每条请求对应的SQL都是并行模式，有可能在某一个时刻并发进程数量、cpu计算资源超过系统可用最大限制，存在数据库不可控宕机的风险。

![7.png](http://cdn.lifemini.cn/dbblog/20200907/6279258368954cc48d56f0022e168339.png)

![8.png](http://cdn.lifemini.cn/dbblog/20200907/5203125dd9884f29be0cba9493f3c867.png)



### 解决方法:

客户反应，之前数据差的挺快的；
是在突然之间，出现的性能问题；
于是通过display_awr语句调取一周之前的语句发现执行计划如图所示。
![9.png](http://cdn.lifemini.cn/dbblog/20200911/f346d773283447a7ba6f95c35d5bf208.png)

可以看到，执行计划中出现了sort unique；
sort unique的表示通过排序取出唯一值，而在后面的执行计划中恰恰缺失了这关键的一步。
推测可能，在收集完统计信息后表中判定不存在重复行，因此走了别的执行计划。
最终决定将统计信息删除，让优化器通过动态采样的方式生成执行计划。
![11.png](http://cdn.lifemini.cn/dbblog/20200911/b1bc69230604445f80e12d827df5d638.png)

实测运行时间，恢复至3.5秒。
![10.png](http://cdn.lifemini.cn/dbblog/20200907/bcbdfe7ff77e463eab8e8f2bd2873b7d.png)



### 案例总结:

开发进行良好的表结构设计是必要的，
否则，有些问题根本无法弥补。