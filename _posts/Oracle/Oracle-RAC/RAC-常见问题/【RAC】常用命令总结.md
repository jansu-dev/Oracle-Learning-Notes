---
title: 【RAC】常用命令解释 
date: 2019-10-29  
categories: Oracle
tags: Oracle-RAC
---



##### RAC中的命令集可根据RAC的架构层次分为：  

- [节点层:osnodes](节点层)
- [网络层:oifcfg](网络层)  
- [集群层:crsctl,ocrcheck,ocrdump,ocrconfig](集群层)  
- [应用层:srvctl,onsctl,crs_stat](应用层)  



### 节点层： 

olsnodes的命令位于/u01/11.2.0/grid/bin目录下，可在Linux中使用find查找olsnodes。
```
[root@core1 ~]# olsnodes -h
Usage: olsnodes [ [-n] [-i] [-s] [-t] [<node> | -l [-p]] | [-c] ] [-g] [-v]
  where
    -n print node number with the node name
    [翻译]-n  打印节点编号和节点名称
    -p print private interconnect address for the local node  
    [翻译]-p 打印私有网址为本地节点
    -i print virtual IP address with the node name 
    [翻译]-i 打印虚拟IP地址和节点名
    <node> print information for the specified node
    [翻译]<node>打印指定节点信息
    -l print information for the local node 
    [翻译] -l 打印本地节点信息
    -s print node status - active or inactive 
    [翻译] -s 打印节点状态{active|inactive}
    -t print node type - pinned or unpinned   
    [翻译] -t 打印节点状态是否被占用
    -g turn on logging  
    [翻译] -g 启动事件记录
    -v Run in debug mode; use at direction of Oracle Support only.  
    [翻译] -v 再详细模式下运行
    -c print clusterware name
    [翻译] -c 打印集群名
```



### 网络层:   

网络层由各网络组件构成，节点IP，节点VIP，节点SCAN    
oficfg命令用来定义和修改网络中网卡的属性
```
[root@core1 crs]# oifcfg
Name:
  oifcfg - Oracle Interface Configuration Tool.
  [翻译]oifcfg - Oracle 接口（网卡）配置工具
Usage:  oifcfg iflist [-p [-n]]
  [解释]iflist列出Oracle网络组件配置信息，-p
  oifcfg setif {-node <nodename> | -global} {<if_name>/<subnet>:<if_type>}...
  oifcfg getif [-node <nodename> | -global] [ -if <if_name>[/<subnet>] [-type <if_type>] ]
  oifcfg delif {{-node <nodename> | -global} [<if_name>[/<subnet>]] [-force] | -force}
                                          [翻译]   <subnet>  = 子网掩码地址

  oifcfg [-help]

  <nodename> - name of the host, as known to a communications network 
  [翻译]<nodename> = 网络中可知并可到达的主机名
  <if_name>  - name by which the interface is configured in the system
  [翻译]<if_name>  = 系统中配置的网卡名
  <subnet>   - subnet address of the interface
  [翻译]<if_type>  = 网卡类型
  <if_type>  - type of the interface { cluster_interconnect | public }
```



### 集群层：

集群层是指以clusterware为核心，组成的软件组合层。   
集群层负责维护群内的共享设备（ASM），DBA可依据集群层提供的完整视图进行需要的调整。    

针对crsctl资源命令（普遍）  ：  

- crsctl   crsctl命令同样存储在/u01/11.2.0/grid/bin/目录下，在环境变量不可用时可直接进入目录使用./crsctl命令进行配置。

针对votedisk管理命令  “

- ocrcheck   
- ocrdump    
- ocrconfig  

##### CRS资源
CRS由CRS，CSS，EVM三个服务组成，每个服务又是由一系列module组成，crsctl允许对每个module进行跟踪，并把跟踪内容记录到日志中。    
[root@rac1 bin]# crsctl lsmodules css  
[root@rac1 bin]# crsctl lsmodules evm  

*  跟踪CSSD模块，并查看日志，需要root用户执行。  
CSSD:后边的内容表示最终的级别：  
```
[root@rac1 bin]# crsctl debug log css "CSSD:1"  
Configuration parameter trace is now set to 1.  
Set CRSD Debug Module: CSSD Level: 1  

[root@rac1 cssd]# pwd  
/u01/11.2.0/grid/log/core1/cssd  
[root@rac1 cssd]# more ocssd.log  
```

#### 维护votedisk
图形安装grid配置Votedisk时，External Redundancy策略只能填写一个Votedisk。   
但可以通过crsctl命令添加多个Vodedisk，这些Votedisk互为镜像以预防Votedisk的单点故障。  
Votedisk使用一种“多数可用算法”来踩就饿是否能够正常使用，即：多个Votedisk必须保持一半以上的Votedisk同时使用，grid才能正常使用。   
当不满足一多数算法条件时，集群会立即宕掉所有节点立即重启。   

操作：添加和删除Votedisk的操作比较危险  

- 必须停止数据库  
- 停止ASM  
- 停止CRS   
- 并且操作时必须使用-force参数。

（1） 查看当前配置           crsctl query css votedisk   
（2） 停止所有节点的CRS      crsctl stop crs   
（3） 添加Votedisk          crsctl add css votedisk /dev/raw/rac1 -force 

##### 注意：即使在CRS关闭后，也必须通过-force参数来添加和删除Votedisk，  且-force参数只有在CRS关闭的场合下使用才安全，  否则会报：Cluter is not a ready state for online disk addition。   

（4） 确认添加后的情况：         crsctl query css votedisk   
（5） 启动CRS                 crsctl start crs  


#### ocr命令系列
grid把所有集群的配置信息放在共享盘柜上存储（OCR Disk）  
集群中，只有master node节点可以对OCR文件进行读写，所有节点都会在内存中保留一份OCR的拷贝   
同时各节点有一个名为OCR Process进程内存中读取内容。   

OCR内容改变时，由Master Node的OCR Process负责同步到其他节点的OCR Process。 
Oracle每4个小时对其做一次备份，并且保留最后的3个备份、前一天、前一周的最后一个备份。  
这个备份由Master Node CRSD进程完成，备份的默认位置是$CRS_HOME\crs\cdata\<cluster_name>目录下。  
每次备份后，备份文件名自动更改最近一次的备份叫作backup00.ocr。


##### ocrdump
该命令以ASCII的方式打印出OCR内容，此命令不能用作OCR的备份恢复只能读操作。 
```
命令格式：ocrdump [-stdout] [filename] [-keyname name] [-xml] 
参数说明： 
-stdout： 把内容打印输出到屏幕上 
Filename：内容输出到文件中 
-keyname：只打印某个键及其子健内容 
-xml：    以xml格式打印输出


示例：把system.css键的内容以.xml格式打印输出到屏幕 
[root@rac1 bin]# ocrdump -stdout -keyname system.css -xml|more 
…… 
这个命令在执行过程中，会在$CRS_HOME\log\<node_name>\client目录下产生日志文件，文件名ocrdump_<pid>.log,如果命令执行出现问题，可以从这个日志查看问题原因。
```

##### ocrcheck   
```
Ocrcheck命令用于检查OCR内容的一致性，命令执行过程会在$CRS_HOME\log\nodename\client目录下产生ocrcheck_pid.log日志文件。 
这个命令不需要参数。 
[root@rac1 bin]# ocrcheck
```

##### ocrconfig  
```
ocrconfig命令用于维护OCR磁盘，安装grid过程中，以External Redundancy冗余方式。  
OCR磁盘最多只能有两个，一个Primary OCR和一个Mirror OCR两个OCR互为镜像放置单点故障。 
[root@rac1 bin]# ocrconfig –help

查看自助备份 
[root@rac1 bin]#./ocrconfig -showbackup 
缺省情况下，OCR自动备份在$CRS_HOME\CRS\CDATA\cluster_name目录下，可通过ocrconfig -backuploc <directory_name> 命令修改到新的目录
```

#### 使用导出导入进行备份恢复
Oracle推荐在对集群做调整时，比如增加，删除节点之前，应该对OCR做一个备份，可以使用export备份到指定文件。   
如果做了replace或者restore等操作，Oracle建议使用cluvfy comp ocr -n all命令来做一次全面的检查。

```
1） 首先关闭所有节点的CRS                 crsctl stop crs 
2） 用root用户导出OCR内容                ocrconfig -export /u01/ocr.exp
3） 重启CRS                            crsctl start crs
4） 检查CRS状态                         crsctl check crs 
5） 破坏OCR内容           dd if=/dev/zero f=/dev/raw/rac1 bs=1024 count=102400 
6） 检查OCR一致性                        ocrcheck 
7） 使用cluvfy工具检查一致性              runcluvfy.sh comp ocr -n all 
8） 使用Import恢复OCR内容                ocrconfig -import /u01/ocr.exp 
9） 再次检查OCR                          ocrcheck 
10）使用cluvfy工具检查                   runcluvfy.sh comp ocr -n all
```

####  移动ocr文件位置

- 实例演示：将OCR从/dev/raw/rac1移动到/dev/raw/raw3上。  

```
1） 查看是否有OCR备份  ocrconfig -showbackup 
如果没有备份，可以立即执行一次导出作为备份  ocrconfig -export /u01/ocrbackup -s online 
2） 查看当前OCR配置    ocrcheck 
3)添加一个Mirror OCR  ocrconfig -replace ocrmirror /dev/raw/raw4 
4)确认添加成功         ocrcheck
5）改变primary OCR位置 ocrconfig -replace ocr /dev/raw/raw3 
确认修改成功： ocrcheck
6）使用ocrconfig命令修改后，所有RAC节点上的/etc/oracle/ocr.loc文件内容也会自动同步了，如果没有自动同步，可以手工的改成以下内容。 
[root@rac1 bin]#more /etc/oracle/ocr.loc 
ocrconfig_loc=/dev/raw/rac1 
Ocrmirrorconfig_loc=/dev/raw/raw3 
local_only=FALSE  
```



### 应用层：
​		应用层就是指RAC数据库了，这一层有若干资源组成，每个资源都是一个进程或者一组进程组成的完整服务，这一层的管理和维护都是围绕这些资源进行的。   

有如下命令:  

- srvctl   
- onsctl   
- crs_stat


#### crs_stat  

##### 查看CRS日志文件

查看报警日志路径show parameter dump可以找到报警日志位置    
/u01/11.2.0/grid/log/core1/crsd/crsd.log
具体安装目录可能不同，grid目录是关键点在grid目录下查找日志文件。   

```shell
SYS@prod1>show parameter cluster_database

NAME                                      TYPE    VALUE
------------------------------------ ----------- ------------------------------
cluster_database                        boolean     TRUE
cluster_database_instances              integer     2
```
cluster_database           -- 查看当前是否为数据库   
cluster_database_instances -- 查看当前数据库的实例数量

##### crsctl stat res -t  

```shell  
[grid@core1 ~]$ crsctl stat res -t

-----------------------------------------------------------------------------
NAME              TARGET  STATE   SERVER                   S  TATE_DETAILS
-----------------------------------------------------------------------------
Local Resources
-----------------------------------------------------------------------------
ora.DATA.dg        
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2
ora.FRA.dg         
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2
ora.LISTENER.lsnr  
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2 
ora.OCR_VOTE.dg    
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2
ora.asm            
                   ONLINE  ONLINE  core1     Started                   
                   ONLINE  ONLINE  core2     Started
ora.gsd            
                   OFFLINE OFFLINE core1     
                   OFFLINE OFFLINE core2
ora.net1.network      
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2
ora.ons            
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2
ora.registry.acfs  
                   ONLINE  ONLINE  core1     
                   ONLINE  ONLINE  core2  
------------------------------------------------------------------------------
Cluster Resources
------------------------------------------------------------------------------
ora.LISTENER_SCAN1.lsnr1 
                    ONLINE  ONLINE  core2
ora.core1.vip1      
                    ONLINE  ONLINE  core1  
ora.core2.vip1      
                    ONLINE  ONLINE  core2         
ora.cvu1            
                    ONLINE  ONLINE  core2               
ora.oc4j1           
                    ONLINE  ONLINE  core1                    
ora.prod.db
        1           ONLINE  ONLINE  core1     Open                
        2           ONLINE  ONLINE  core2     Open                
ora.scan1.vip1      
                    ONLINE  ONLINE  core2              
```

##### crsctl state res -t表示以表格的当时查询CRS资源状态

```
Crs_stat这个命令用于查看CRS维护的所有资源的运行状态   
如果不带任何参数显示所有资源的概要信息。   
每个资源显示属性：资源名称，类型，目录，资源运行状态等。 

[root@rac1 bin]# crs_stat查看指定资源的状态
1） 查看指定资源状态 crs_stat ora.rac2.vip   
2） 使用-v选项，查看详细内容，这时输出多出4项内容，分别是允许重启次数，已执行重启次数，失败阀值，失败次数。   
crs_stat -v ora.rac2.vip   
3） 使用-p选项查看更详细内容 
crs_stat -p ora.rac2.vip
4） 使用-ls选项，可以查看每个资源的权限定义，权限定义格式和Linux一样。 
[root@rac1 bin]#./crs_stat -ls
```
1. crsctl check crs命令查看RAC集群状态，四个服务是否全部正常运行。在刚开启集群时，四个服务可能在不同的实验机上，依据机器的性能不同，开始时间可能也不同。  
2. 四个服务在11g中，四个服务的开启顺序是CSSD、CRSD、HA、EVM。  

##### 与VIP相关
在RAC集群中手动切换到其他VIP   
crsctl relocate resource ora.rac2.vip


##### onsctl  

用于管理配置ONS(Oracle Notification Service)，ONS是grid实现FAN Event Push模型的基础。
在传统模型中，客户端需要定期检查服务器来判断服务端状态，本质上是一个pull模型，Oracle 10g引入了一个全新的PUSH机制–FAN(Fast Application Notification),当服务端发生某些事件时，服务器会主动的通知客户端这种变化，这样客户端就能尽早得知服务端的变化。而引入这种机制就是依赖ONS实现， 在使用onsctl命令之前，需要先配置ONS服务。


##### ons配置路径

注意：在RAC环境中，需要使用$CRS_HOME下的ONS,而不是$ORACLE_HOME下面的ONS。 配置文件在$CRS_HOME\opmn\conf\ons.config. 
[root@rac1 conf]# more /u01/app/oracle/product/crs/opmn/conf/ons.config 

##### ons配置参数说明： 
Localport:本地监听端口，这里本地特指：127.0.0.1这个回环地址，用来和运行在本地的客户端进行通信 
Remoteport：远程监听端口，除127.0.0.1以外的所有本地IP地址，用来和远程的客户端进行通信。 
Loglevel:Oracle允许跟踪ONS进程的运行，并把日志记录到本地文件中，这个参数用来定义ONS进程要记录的日志级别，从1-9，缺省值是3. 
Logfile:这个参数和loglevel参数一起使用，用于定义ONS进程日志文件的位置，缺省值是$CRS_HOME\opmn\logs\opmn.log 
nodes和useocr:这两个参数共同决定本地的ONS daemon要和哪些远程节点上的ONS daemon进行通信。 


Nodes参数值格式   
Hostname/IP:port[hostname/ip:port] 
如：useoce=off Nodes=rac1:6200,rac2:6200 

useocr参数值为on/off   
1. ON：将信息保存在OCR中  
2. OFF：信息取nodes中的配置   
3. 单实例要把useocr设置为off


##### 配置ons：
可以直接编译ONS的配置文件来修改配置，也可以通过racgons命令进行（必须是root用户）配置，   
如果用oracle用户来执行，不会提示任何错误，但也不会更改任何配置。   

添加配置命令： Racgons add_config rac1:6200 rac2:6200 
删除配置命令： Racgons remove_config rac1:6200 rac2:6200


##### onsctl命令解析
使用onsctl命令可以启动，停止，调试ONS，并重新载入配置文件，其命令格式如下： 
[root@rac1 bin]#./onsctl 

ONS进程运行，并不一定代表ONS正常工作，需要使用ping命令来确认。 
1） 在OS级别查看进程状态。 
[root@rac1 bin]#ps -aef|grep ons 
2） 确认ONS服务的状态 
[root@rac1 bin]#./onsctl ping 
3） 启动ONS服务 
[root@rac1 bin]#./onsctl start
4）使用debug选项，可以查看详细信息，其中最有意义的就是能显示所有连接。 
[root@rac1 bin]#./onsctl debug 
[root@rac1 bin]#


##### srvctl  

工具srvctl可以操作下面的几种资源：  

- Database  
- Instance  
- ASM  
- Service  
- Listener  
- Node Application  
  - Node application又包括GSD，ONS，VIP。 

 以上资源除了使用srvctl工具管理外，某些资源还有自己独立的管理工具   
 ONS可以使用onsctl命令进行管理  
 Listener可以通过lsnrctl管理   


##### srvctl命令解析
```
config查看配置：

1）查看数据库配置 
–不带任何参数时，显示OCR中注册的所有数据库 
[root@rac1 bin]# srvctl config database 

-a选项查看配置的详细信息 
[root@rac1 bin]# srvctl config database -d raw -a 

2）查看Node Application的配置 
–不带任何参数，返回节点名，实例名和$ORACLE_HOME 
[root@rac1 bin]#./srvctl config nodeapps -n rac1

–使用-a选项，查看VIP配置 
[root@rac1 bin]#./srvctl config nodeapps -n rac1 -a

–使用-g选项，查看GSD： 
[root@rac1 bin]#./srvctl config nodeapps -n rac1 -g 

–使用-s选项，查看ONS: 
[root@rac1 bin]#./srvctl config nodeapps -n rac1 -s 

–使用-l选项，查看Listener: 
[root@rac1 bin]# ./srvctl config nodeapps -n rac1 -l 
Listener exists. 

3)查看Listener. 
[root@rac1 bin]#./srvctl config listener -n rac1 
[root@rac1 bin]#./srvctl config listener -n rac2 

4)查看ASM 
[root@rac1 bin]#./srvctl config asm -n rac1 
[root@rac1 bin]#./srvctl config asm -n rac2 

5） 查看Service 
–查看数据库所有service配置 
[root@rac1 bin]#./srvctl config service -d raw -a 
–查看某个Service配置 
[root@rac1 bin]#./srvctl config service -d raw -s dmm 
–使用-a选项，查看TAF策略 
[root@rac1 bin]#./srvctl config service -d raw -s dmm -a


##### 使用add添加对象：

一般情况下，应用层资源都是在图形界面的帮助下注册到OCR中的，比如VIP，ONS实在安装最后阶段创建的数据库，ASM是执行DBCA的过程中自动注册到OCR中的  
Listener是通过netca工具。 但是有些时候需要手工把资源注册到OCR中。    

使用add命令手动添加LISTENER： 
1） 添加数据库 
[root@rac1 bin]#./srvctl add database -d dmm -o $ORACLE_HOME 
2) 添加实例 
[root@rac1 bin]#./srvctl add instance -d dmm -n rac1 -i dmm1 
[root@rac1 bin]#./srvctl add instance -d dmm -n rac2 -i dmm2 
3)添加服务，需要使用4个参数 
- s :服务名 
- r：首选实例名 
- a：备选实例名 
- P：TAF策略，可选值为None（缺省值），Basic，preconnect。 
[root@rac1 bin]# ./srvctl add service -d dmm -s dmmservice -r rac1 -a rac2 -P BASIC 
4)确认添加成功 [root@rac1 bin]# ./srvctl config service -d dmm -s dmmservice -a


使用enable/disable启动/禁用对象：
缺省情况下数据库，实例，服务，ASM都是随着CRS的启动而自启动的，处于维护需要可手动关闭某特性。  

1）配置数据库随CRS的启动而自动启动 
– 启用数据库的自启动： 
[root@rac1 bin]#./srvctl enable database -d raw 
– 查看配置 
[root@rac1 bin]# ./srvctl config database -d raw -a 
– 禁止数据库在CRS启动后自启动，这时需要手动启动 
[root@rac1 bin]# ./srvctl disable database -d raw 
2） 关闭某个实例的自动启动 
[root@rac1 bin]# ./srvctl disable instance -d raw -i rac1 
[root@rac1 bin]# ./srvctl enable instance -d raw -i rac1 
– 查看信息 
[root@rac1 bin]# ./srvctl config database -d raw -a 
3)禁止某个服务在实例上运行 
[root@rac1 bin]# ./srvctl enable service -d raw -s rawservice -i rac1 
[root@rac1 bin]# ./srvctl disable service -d raw -s rawservice -i rac1 
– 查看 
[root@rac1 bin]# ./srvctl config service -d raw -a 
dmm PREF: rac2 AVAIL: rac1 TAF: basic


##### remove删除对象：  

使用remove命令删除的是对象在OCR中的定义信息，对象本省比如数据库的数据文件等不会被删除，以后随时可以使用add命令重新添加到OCR中。 
1)删除Service，在删除之前，命令会给出确定提示 
[root@rac1 bin]# ./srvctl remove service -d raw -s rawservice 
2）删除实例，删除之前同样会给出提示 
[root@rac1 bin]# ./srvctl remove instance -d raw -i rac1 
3）删除数据库 
[root@rac1 bin]# ./srvctl remove database -d raw


##### 跟踪srcctl
Oracle 10g中，跟踪srvctl只要设置srvm_trace=true（OS环境变量）即可，   
这个命令的所有函数调用都会输出到屏幕上，可以帮助用户进行诊断。 
[root@rac1 bin]# export SRVM_TRACE=TRUE 
[root@rac1 bin]# srvctl config database -d raw


##### 恢复实验
前提：设OCR磁盘和Votedisk磁盘全部破坏，并且都没有备份。   
解决：重新初始话OCR和Votedisk（最简单）  

具体操作如下：
```
1) 停止所有节点的Clusterware Stack
Crsctl stop crs;
2) 分别在每个节点用root用户执行$CRS_HOME\install\rootdelete.sh脚本
3) 在任意一个节点上用root用户执行$CRS_HOME\install\rootinstall.sh脚本
4) 在和上一步同一个节点上用root执行$CRS_HOME\root.sh脚本
5) 在其他节点用root执行行$CRS_HOME\root.sh脚本
6) 用netca命令重新配置监听，确认注册到Clusterware中

```
#crs_stat -t -v 
到目前为止，只有Listener，ONS,GSD,VIP注册到OCR中，还需要把ASM， 数据库都注册到OCR中。

向OCR中添加ASM
#srvctl add asm -n rac1 -i +ASM1 -o /u01/app/product/database 
#srvctl add asm -n rac2 -i +ASM2 -o /u01/app/product/database
启动ASM
#srvctl start asm -n rac1 
#srvctl start asm -n rac2 
若在启动时报ORA-27550错误。是因为RAC无法确定使用哪个网卡作为Private Interconnect，解决方法：在两个ASM的pfile文件里添加如下参数： 
+ASM1.cluster_interconnects=’10.85.10.119′ 
+ASM2.cluster_interconnects=’10.85.10.121′
手工向OCR中添加Database对象
#srvctl add database -d raw -o /u01/app/product/database
添加2个实例对象
#srvctl add instance -d raw -i rac1 -n rac1 
#srvctl add instance -d raw -i rac2 -n rac2
修改实例和ASM实例的依赖关系
#srvctl modify instance -d raw -i rac1 -s +ASM1 
#srvctl modify instance -d raw -i rac2 -s +ASM2
启动数据库
#srvctl start database-d raw 
若也出现ORA-27550错误。也是因为RAC无法确定使用哪个网卡作为Private Interconnect，修改pfile参数在重启动即可解决。 
SQL>alter system set cluster_interconnects=’10.85.10.119′ scope=spfile sid=’rac1′; 
SQL>alter system set cluster_interconnects=’10.85.10.121′ scope=spfile sid=’rac2′;

srvctl +<command:status,start,stop,config,modify,relocate>+<object:database,service,instance,nodeapps> + <option: -i ,-d,-s,-n>

```
##### scan状态及配置  

```
[root@core1 ~]# srvctl status scan
SCAN VIP scan1 is enabled
SCAN VIP scan1 is running on node core2

[root@core1 ~]# srvctl config scan
SCAN name: 192.168.9.89, Network: 1/192.168.9.0/255.255.255.0/eth0
SCAN VIP name: scan1, IP: /192.168.9.89/192.168.9.89
```
使用srvctl status scan命令查看scan的状态及配置信息。    


启动关闭crs相关进程  crsctl {start|stop} resource ora.cssd  



#### 磁盘组相关

##### 查看磁盘组使用情况 
1. 方法一：
```
[grid@core1 ~]$ asmcmd lsdg
Can not create path /u01/11.2.0/grid/log/diag/asmcmd/user_grid/core1/alert 
Can not create path /u01/11.2.0/grid/log/diag/asmcmd/user_grid/core1/trace 
State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  NORMAL  N         512   4096  1048576     10260             
5978                0            2989              0             N  DATA/

MOUNTED  NORMAL  N         512   4096  1048576      8204     
5104                0            2552              0             N  FRA/

MOUNTED  NORMAL  N         512   4096  1048576      3105                      
2179              1035             572             0             Y  OCR_VOTE/
```
2. 方法二：
```
SYS@+ASM1>select name, state, total_mb, free_mb from v$asm_diskgroup;

NAME             STATE       TOTAL_MB  FREE_MB
------------------------------ ----------- ---------- ----------
DATA             MOUNTED    10260     5978
FRA              MOUNTED     8204     5104
OCR_VOTE         MOUNTED     3105     2179
```

##### 查看磁盘位置
```
SYS@+ASM1>col name for a20;
SYS@+ASM1>col path for a20;
SYS@+ASM1>select name,path from v$asm_disk;

NAME         PATH
-------------------- --------------------
                      /dev/raw/raw11
                      /dev/raw/raw9
                      /dev/raw/raw10
OCR_VOTE_0001         /dev/raw/raw2
DATA_0000             /dev/raw/raw5
DATA_0001             /dev/raw/raw6
FRA_0000              /dev/raw/raw7
OCR_VOTE_0000         /dev/raw/raw1
OCR_VOTE_0002         /dev/raw/raw3
FRA_0001              /dev/raw/raw8

10 rows selected.
```


##### 检查表决盘信息 
```
[root@core1 ~]# crsctl query css votedisk
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   8e156fca11014f4dbf5e092263b76dd1 (/dev/raw/raw1) [OCR_VOTE]
 2. ONLINE   8fbd8708d15a4fa0bf876c963fa011a3 (/dev/raw/raw2) [OCR_VOTE]
 3. ONLINE   43631102a2804f93bfbc5be26cf4c2da (/dev/raw/raw3) [OCR_VOTE]
Located 3 voting disk(s).
```

##### 查看与CRS服务有关的进程   
ps -ef | grep crs  
```shell
[root@core1 ~]# ps -ef |grep crs
root   2973  1     0 06:51 ?      00:03:34 /u01/11.2.0/grid/bin/crsd.bin reboot
root   31305 30074 0 13:42 pts/0  00:00:00 grep crs
[root@core1 ~]# 
```



### 参考文章：

- [参考文章1—Oracle RAC管理及维护命令详解](https://www.cnblogs.com/lhdz_bj/p/9110526.html)
-  [参考文章2—oracle RAC常用命令](https://blog.csdn.net/debimeng/article/details/79614815)
