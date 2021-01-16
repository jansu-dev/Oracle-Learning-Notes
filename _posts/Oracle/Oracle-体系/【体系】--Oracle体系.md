---
title: 【oracle】--体系再整理--Oracle的架构组成
date: 2019-11-30 19:21:20
tags:
---



### 文章原因   

最近突发奇想，想在整理一般Oracle的体系知识。不得不说Oracle在数据库当面真的是设计最严谨的集中式传统数据库。值得深入了解学，也想借此机会对我以往的知识加以巩固。



### Oracle实例

##### Oracle 11g和12c两个版本在实例上的区别  


首先我们用两图看看二者的区别：  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在Oracle 11g版本中，采用单实例单库的对应方式。
instance（实例）由background process(后台进程)和SGA（公共内存空间）两大部分组成，  SGA中存放实例运行过程中所需要的信息，server process（服务进程）和background process（后台进程）共同去处理这个公共内存区域的信息，完成数据库的任务。   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;举例：就像人写作业一样，人往纸上写作业，人就是一个进程（后台进程或用户进程，统称某进程），而写出来的一本作业就是也可以被老师（某进程）批改或家长（某进程）查阅。内存中的信息，当然是一掉电所存储的的信息便不复存在，使用可永久存储的作业本来比喻SGA内存信息有可能不太恰当，但是除是否可永久存储区别外，其运作道理是相同的。   

##### 这oracle server中内存空间需要注意：  
1. SGA和PGA构成了Oracle server所有的内存空间   
2. 一个实例仅有一个SGA  
3. SGA为所有session共享  
4. SGA声明周期为，随实例启动时(startup）产生，实例关闭时（shutdown）关闭。  


12c版本中Multitenant Architecture,翻译过来就是多租户架构。Multi（多）tenant（租户）。12c版本在实例层将系统SGA划分为CDB与PDB模式，CDB（Container Database）容器库下属多个PDB（Plugg Database）插件库，实现了一个主库多个可插拔的插件库。  
Instance（实例)仍然是由数据库



### 参考文章

