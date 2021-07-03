---
title:[TiDB]--数据迁移工具汇总
date:2020-12-29
---



#### [TiDB]--数据迁移工具汇总

- [Data migration工具简介](Data migration工具简介)
- [TiDB数据迁移的方式](TiDB数据迁移的方式)
- [TiDB提供得分5套工具汇总](TiDB提供得分5套工具汇总)
- [如何优化BRIE](如何优化BRIE)
- [BR备份恢复原理](BR备份恢复原理)
- [参考文章](参考文章)



### Data migration工具简介

什么是数据迁移 : 数据迁移就是将数据从一个地方前有到另一个地方；



### TiDB数据迁移的方式

迁移的方式可以分为增量复制与全量复制；

![image.png](http://cdn.lifemini.cn/dbblog/20201228/3431376504124d10a7b47b4daadceb4a.png)

- **全量复制**

- **增量复制**

获取binlog数据进行导入；



### TiDB提供得分5套工具汇总

![image.png](http://cdn.lifemini.cn/dbblog/20201228/417e1fc1599247a9ab32a3d53fc39341.png)

Backup & Restore(BR)

从TiKV快速的备份sst文件，类似于Oracle的expdp一样的物理导出到文件系统；
因为TiDB支持一些Cloud Native的场景，因此也可以存储到S3等环境。

- **Dumpling**

通过导出将SQL数据组合成SQL语句的逻辑格式或CSV等第三方格式的数据文件，在由其他数据库识别此类文件的方式导入，从而实现数据迁移。

![image.png](http://cdn.lifemini.cn/dbblog/20201228/f6e704ee71424ac8833765dc53c465aa.png)

- **Lighting**

Backend 导入

local Backend导入

- **TiCDC**

内部原理是同通过监测TiDB的数据变更事件，进而同步到Kafaka、MySQL等下游数据接受软件；

- **DM**
  既支持全量，有支持增量的方式，是TiDB一整套的数据迁移方案；
  DM的迁移原理也是捕获TiDB的binlog日志，实现数据迁移。



### 如何优化BRIE

优化的目标：BRIE中的迁移工具均是要迁移GB~TB级别的数据，因此首要目标便是迁移的**速度**；

优化的方式：

1.并行(parallization):能并行处理的步骤尽量并行处理；
2.批量(Batiching)：如果遇到不能并行处理的步骤，尽量采用批量处理的方式处理；
3.解决错误(transient errors)：因为数据量较大，所以如果遇到错误，可以采用多次不断尝试的方式解决。



### BR备份恢复原理

- **BR的备份过程**

![image.png](http://cdn.lifemini.cn/dbblog/20201228/845fe9f3fd83439d99db72557df3285e.png)

1.BR的备份不是由BR这个工具操作的，而是由BR给每个TIKV leader发送一个备份信息，具体的备份由TiKV节点基于Regin实现备份操作；
2.TiKV节点会扫描对应Regin的所有键值对，并收集到备份的内存的memtable或SST file中，此时的SST file文件是暂时存在文件系统中的；
3.上传SST File备份文件到终端存储时，如果数据还处于内存中时直接从内存上传到终端系统（如：S3、云文件系统等），如果已经落盘到文件系统的SSt file就将SST file上传到终端文件系统。

- **BR的恢复过程**

![image.png](http://cdn.lifemini.cn/dbblog/20201228/ed3812a5aba94719afdac381e94b69a8.png)

1.在BR的写入阶段，首先使用16线程并行的执行CREATE TABLE TABLE_NAME IF NOT EXIST命令；
2.停止PD Server的调度功能，防止PD Server将正在恢复的Regin调度到其他TiKV节点，以减小无谓的开销；
3.TiKV计入“import mode”模式，通过停止write stall trigger和compaction trigger两个功能，停止SST file文件合并，增加读方法的方式实现更快的恢复；
4.因为在备份的时候，是基于一个Regin备份到一个SST文件实现的，因此最快的方式是将一个SST写入到一个Regin中。相应的优化操作为通过PD Server搜集所有当前数据库的所有SST files的边界,通知PD Server将每一个备份的SST file至少执行“Batch Split”操作打散为1个SST file的多个SST file。**注意**:这里分发到Regin Leader节点，因为TiKV进行数据写入的时候是需要同故宫Raft协议实现分布式一致性，所以应当分发至不同Regin Leader；

5.在还没有数据的时候执行一个scatter操作将打散的SST file（也就是Regin）均匀的分发到不同的Regin leader节点。因为每一个备份的SST file还是存在于原来的Store上面的，所以需要在打散之后由PD Server调度至不同TiKV节点，防止未做此操作后还需要进行的rebalance操作；

6.每个Regin并行的在每个TiKV节点执行restore。



### 参考文章

- [**Bilibili分享-High Performance TiDB】Lesson 12：生态工具优化**](https://www.bilibili.com/video/BV1D5411L7z5)