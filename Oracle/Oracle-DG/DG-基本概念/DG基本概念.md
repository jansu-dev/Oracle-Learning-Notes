### **DG配置方式**
DG由一个主库和若干备库组成，主备库之间以网络连接。
主备库之间相互沟通，不受仅能使用本地局域网络限制。
**例如**：你可以在“北京”和“上海”两地的两台同版本Oracle数据库之间搭建一个DG。



### **物理备用数据库**

在磁盘上的数据库结构与主数据库在块对块的基础上相同,提供与主库物理上相同的副本。
数据库模式(包括索引)是相同的。物理备用数据库通过Redo Apply与主数据库保持同步，
Redo Apply恢复从主数据库接收到的重做数据，并将redo应用于物理备库。

**Active Data Guard(ADG)**
从Oracle数据库11g版本1(11.1)开始，物理备库可以在**只读模式打开时**接收和应用重做redo。
因此，可以同时使用物理备用数据库进行数据保护和报告。
![物理备库](http://cdn.lifemini.cn/dbblog/20210115/2a065e73ddcf478c9954bc48af0a172e.png)





### **逻辑备用数据库**

数据的物理组织和结构可能不同，如：平台异构。主库与备库逻辑信息相同。逻辑备库通过SQL Apply与主库同步，后者将从主数据库接收的重做中的数据转换为SQL语句，然后在备用数据库上执行SQL语句。

<div style="color:red">将主库的redo log传递到备库后，再利用logminer 的工具，从redo log中解析出sql语句，在备库执行，保证和主库同步。（主库和备库可以是不同的环境，备库可以处于读写状态）逻辑备库可以看作是一个单独的库，数据库名和DBID和主库都不一样。（物理的备库和主库的数据库名和DBID都是同样的。）</div>

除了灾难恢复外，用户可以随时访问用于查询和报告目的的逻辑备库。使用逻辑备库，可升级Oracle数据库软件和补丁集，几乎不需要停机。因此，可以同时使用逻辑备用数据库进行数据保护、报告和数据库升级。
![逻辑备库](http://cdn.lifemini.cn/dbblog/20210115/5e0e0837d27b42b68d6e52860583637e.png)





### **备用数据库快照**

##### 快照备用数据库是完全可更新的备用数据库。

与物理或逻辑备库一样，快照备库从主库接收并归档redo数据。
与物理或逻辑备库不同，快照备用数据库不应用它接收到的重做数据。
快照备库接收到的重做数据在将快照备库转换回物理备库之前不会应用，
转换物理备库之前，将首先丢弃对快照备库所做的任何本地更新。