---
title: oracle备份恢复之recover database的四条语句区别
date: 2019-9-3
categories: Oracle
tags: Oracle常见问题
---


#

##### 四条语句：

1  recover database using backup controlfile
2  recover database until cancel
3  recover database using backup controlfile until cancel;
4  recover database until cancel using backup controlfile;



#### 本文主要介绍以下四种恢复方式的含义与区别：

1. ##### recover database using backup controlfile

 如果丢失当前控制文件，用冷备份的控制文件恢复的时候  用来告诉oracle，不要以controlfile中的scn作为恢复的终点；

2. ##### recover database until cancel

 如果丢失current/active redo的时候，手动指定终点。

3. ##### recover database using backup controlfile until cancel;

如果丢失当前controlfile并且current/active redo都丢失，  
会先去自动应用归档日志,可以实现最大的恢复；

4. ##### recover database until cancel using backup controlfile;

  与recover database using backup controlfile until cancel效果一样  

 Oracle会以当前controlfile所纪录的SCN为准，利用archive log和 redo log的redo entry, 把相关的datafile 的 block恢复到“当前controlfile所纪录的SCN”

某些情况下，Oracle需要把数据恢复到比当前controlfile所纪录的SCN还要靠后的位置  
（如：control file是backup controlfile , 或者 controlfile是根据trace文件建立的。）  
这时候需用using backup controlfile恢复就不会受“当前controlfile所记录的SCN”的限制。  



### 结论：

1、适用于restore旧的控制文件，且归档日志和cuurrent/active redo都没有丢失情况。如果一切归档日志和在线日志完好，可以不丢失数据。类似于recover database
2、当前控制文件未丢失（不需要restore旧的控制文件），此时有归档日志或者current/active log有丢失情况下，则终止。最大可能恢复数据
3、4:我在oracle 10.2.0.4环境下测试效果是相同的，即适用于restore旧的控制文件，在恢复到控制文件备份那刻后，系统会提示应用控制文件备份后的归档日志，如果没有则停止。也是最大可能的恢复数据。