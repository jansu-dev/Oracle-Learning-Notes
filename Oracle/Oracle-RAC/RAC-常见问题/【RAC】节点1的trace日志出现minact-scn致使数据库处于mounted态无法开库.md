### 文章概览

- [数据库信息](#数据库信息)
- [是什么？](#是什么？) 
- [为什么？](#为什么？)
- [怎么做？](#怎么做？)  
- [参考文章](#参考文章)



### 数据库信息

1. 数据库版本:Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit
2. RAC节点数量：2节点
3. 操作系统版本：Red Hat Enterprise Linux Server release 5.5 (Tikanga)
4. 内存管理机制：大页存储



### 是什么？  

<p style="color:red">-----两节点RAC，节点1的trace日志出现minact-scn致使数据库处于mounted态无法开库-----</p>
本次操作为对Oracle11g两节点RAC所在服务器扩充内存， 

首先node2节点数据库正常关闭，但因前些天更改大页存储时，  
出现过关机node2服务器node1也随之关闭的现象，
为降低服务器突然宕机损坏磁盘或出现Oracle实例恢复等风险。


本次在node2关库关集群修改完毕内存配置信息后，
马上将node1内存参数修改完毕，才敢在node2节点所在服务器做shutdown操作。


<p style="color:red">------出现问题------</p>
node2上正常关闭扩充内存后，本次没出现node2服务器关机牵连node1服务器突然宕机现象，
node2上正常起集群、起库后，在node1上做shutdown immediate操作，准备在node1上做更换内存操作，
但此时，node1出现20分钟无法shutdown immediate的现象，trace日志如下。
```
Active call for process 13883 user 'grid' program 'oracle@XXXXX'
SHUTDOWN: waiting for active calls to complete.
Tue Apr 21 21:23:32 2020
Thread 1 advanced to log sequence 346823 (LGWR switch)
  Current log# 11 seq# 346823 mem# 0:
   +DATA2/XXXX/onlinelog/group_11.270.829088701
Tue Apr 21 21:26:17 2020
All dispatchers and shared servers shutdown
ALTER DATABASE CLOSE NORMAL
Tue Apr 21 21:26:29 2020
Auto-tuning: Shutting down background process GTX1
Tue Apr 21 21:28:47 2020
...............
...............
Tue Apr 21 21:29:32 2020
Instance shutdown complete
```
可以看到 

<p style="color:red">Active call for process 13883 user 'grid' program 'oracle@XXXXX'</p>

可能是node2加内存后的开库操作，导致node1上进程号为13883的进程阻止node1数据库关闭，
在我刚要kill掉13883进程时，Auto-tuning开始自动调优关库。
在node1服务器加完内存后，数据库出现一直卡在mounted阶段无法开库情况，日志显示如下。


```
MTTR advisory is disabled because FAST_START_MTTR_TARGET is not set
Tue Apr 21 23:32:46 2020
SMON: enabling cache recovery
Tue Apr 21 23:32:47 2020
minact-scn: Inst 1 is a slave inc#:4 mmon proc-id:15972 status:0x2
minact-scn status: grec-scn:0x0000.00000000 
gmin-scn:0x0000.00000000 
gcalc-scn:0x0000.00000000
```



### 为什么？  

<p style="color:red">-----本人猜测，无法佐证-----</p>
因为node2节点先开机node1节点后开机，此外，之前node1出现一直无法关机最后由数据库Auto-tuning自动关库现象，
本人猜测数据库Auto-tuning本身也是kill掉13883进程才实现的关库。


当node1加好内存后，开机仍需要实例恢复，此时实例恢复需要undo中内容依据redo重放，
因为node2率先开机，可能存在node2节点已经执行了部分node1需要实例恢复时恢复的事务，
从而影响node1的undo表空间无法online，undo信息无法依据redo信息重放，最终无法开库。

可以看到在node2上关库后，node1的trace日志显示<p style="color:red">Successfully onlined Undo Tablespace 2</p>
```
minact-scn: Inst 1 is a slave inc#:8 mmon proc-id:20782 status:0x2
minact-scn status: grec-scn:0x0000.00000000 
gmin-scn:0x0000.00000000 gcalc-scn:0x0000.00000000
Wed Apr 22 00:15:03 2020
[20870] Successfully onlined Undo Tablespace 2.
Undo initialization finished serial:0 start:2019894
 end:2662754 diff:642860 (6428 seconds)
Verifying file header compatibility for 11g tablespace encryption..
Verifying 11g file header compatibility for tablespace encryption 
completed

```


中间参考过，[文章Bug 11891463](https://blog.csdn.net/xhailing/article/details/12912317)说matelink上有说这是Oracle 11g以后的一个BUG，但是仔细观察错误现象又不太一样；

<p style="color:red">Bug 11891463所反映的现象是每5分钟便在trace日子中打印一次minact-scn master-status；</p><p style="color:red">
本文出现问题为，trace文件中输出内容卡在第一次输出minact-scn master-status位置无法进行；</p>
因为是生产库，在node2节点做完shutdown immediate操作node1顺利起库后便没有做多余操作。



### 怎么做？

如果在生产库上遇到，某一节点一直无法开库，在不影响业务并且trace日志出现本文所示minact-scn信息的情况下，可尝试如下操作。
1. 首先，node2节点执行shutdown immediate操作,使得node1可以online自己的undo表空间；
2. 其次，观察node1是否顺利起库及trace日志打印信息。



### 参考文章   
 - [Bug 11891463](https://blog.csdn.net/xhailing/article/details/12912317)





> 如果发现本文章有待商榷之处，
> 可发送邮件至coresu@icloud.com，
> 本人将及时改正。
