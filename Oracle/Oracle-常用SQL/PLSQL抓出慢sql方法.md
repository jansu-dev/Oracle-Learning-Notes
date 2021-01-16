#### 文章概览
- [是什么？](#是什么？) 
- [为什么？](#为什么？)  
- [怎么做？](#怎么做？)  
- [参考文章](#参考文章)



### 是什么？  

使用PL/SQL工具抓取执行慢SQL发现拖慢业务原因，是一种简单、高效、易用的一种方式。



### 为什么？  

在日常的数据库运维工作中，经常会出现原本参与业务的表数据量很小，原sql运行的很快，但随着业务数据量的增长，sql的运行时间会逐渐拖长，运行效率会逐渐拖慢。   
<font color="red">当公司其他部门将问题反馈到DBA时，作为DBA的我们如何抓住问题源，快速解决问题恢复业务呢？</font>  
工作中导致数据库慢的原因有很多，但Oracle发展到现在12c的版本已经非常自动化、服务器的硬件性能也不再是难题，因此其中很多问题都来自于sql与业务的不匹配。  
可以使用本文提供的方式快速在工作中，定位到执行比较慢的SQL，再依据经验针对SQL做出优化。



### 怎么做？


##### 1. 登陆后，进入sessions模块
![进入session](http://cdn.lifemini.cn/dbblog/20210115/bdac71051c6048fcbad2832b04f14804.png) 

PL/SQL Developer工具将我们的操作已经做的非常自动化了。  
首先登陆PL/SQL Developer客户端，具体登陆步骤就不在这里介绍了，有不会使用PL/SQL  Developer可以自行百度，本文也会提供给大家PL/SQL Developer官方使用手册浏览入口。   

出现截图所示页面后，点击红框圈出的<font color="red">Tools</font>会弹出截图中的选项菜单,再点击<font color="red">sessions</font>模块。  

##### 2. 抽取当前活跃session会话  
![活跃session](http://cdn.lifemini.cn/dbblog/20210115/1c62b3414ae04dd3ade8d483225fce9f.png)   
点击filter选择框中不同session会话状态进行筛选，点击active sessions获取活跃会话。  
如图所示，因截图为测试库未运行JOB和sql，所以仅有一个PL/SQL客户端连接活跃会话。

##### 3. 对active sessions排序  
![filter-session](http://cdn.lifemini.cn/dbblog/20210115/641fa3dd9a7f444c83b6eb3ea85fe666.png)   
可点击蓝色按钮，对last call et字段依据升序或降序排序。我们采用降序排序找出当今所有active session中执行时间最长的session。   

- 注意：此时的active session可能是用来跑JOB的session也，有可能是直接运行SQL语句的session。
    + 如果是直接运行SQL语句的session，last call st为10几秒，那么此条SQL一定出现了问题。
    + 如果是运行JOB的session，因为JOB一组存储过程集合，故执行时间较长，需要进入第4步判定。

##### 4. 查看执行时间最长session的SQL详细信息  
![info-session](http://cdn.lifemini.cn/dbblog/20210115/b234670ceb5246e5ba51aa3e1df44c1a.png)    
在SQL sessions的下半部分显示了所选会话的详细信息，包含(Cursors)游标、(SQL Text)当前运行SQL、(Statics)统计信息、(Locks)锁、(SQL Monitor)SQL监控。   

<font color="red">可以通过不断点击图片左上角绿色选框刷新状态，如果有某一条sql长时间没有运行结束，既：在SQL Text中长时间未改变，很有可能此条SQL就是问题源！</font>



### 参考文章   

 - [参考文章1-PL/SQL中session使用](https://jingyan.baidu.com/article/3ea51489eb65b152e61bba8b.html)  
 - [参考文章2-PL/SQL官方使用手册](../Common File/PLSQLDeveloper11.0用户指南（中文版带目录书签）.pdf)



