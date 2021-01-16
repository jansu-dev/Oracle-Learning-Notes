---
title: NFS权限问题--root用户出现Permission denied
date: 2019-10-22 
categories: Oracle
tags: Oracle-RAC
---



# exports文件简介

exports是NFS服务的配置文件，可在此文件内部配置NFS相关共享目录路径、IP地址、权限参数。



### /etc/exports书写格式

| 第一列 | 第二列 | 第三列 |
| :---: | :---: | :---: |
| 欲共享出去的本地目录 | 可访问的host | 读写等共享参数 |
| /home/test | 192.168.56.9 | (rw,sync) |

第三列：共享参数参考
 ro                       只读访问   
 rw                       读写访问   
 sync                     所有数据在请求时写入共享   
 async                    NFS在写入数据前可以相应请求   
 secure                   NFS通过1024以下的安全TCP/IP端口发送   
 insecure                 NFS通过1024以上的端口发送   
 wdelay                   如果多个用户要写入NFS目录，则归组写入（默认）   
 no_wdelay         如果多个用户要写入NFS目录，则立即写入，当使用async时，无需此设置。   
 Hide                     在NFS共享目录中不共享其子目录   
 no_hide                  共享NFS目录的子目录   
 subtree_check     如果共享/usr/bin之类的子目录时，强制NFS检查父目录的权限（默认）   
 no_subtree_check         和上面相对，不检查父目录权限   
 all_squash               共享文件的UID和GID映射匿名用户anonymous，适合公用目录。   
 no_all_squash            保留共享文件的UID和GID（默认）   
 root_squash              root用户的所有请求映射成如anonymous用户一样的权限（默认）   
 no_root_squash           root用户具有根目录的完全管理访问权限   
 anonuid=xxx              指定NFS服务器/etc/passwd文件中匿名用户的UID



## 实验

```shell
在节点core1上 
[root@core1 u01]# vi /etc/exports 
/u01/arch1 192.168.9.99(rw,sync)


重新扫描/etc/exports文件
exportfs -av


在节点core2上挂载并测试
[root@core2 ~]mount core2:/u01/arch1 /u01/arch1
[root@core2 ~]# df
Filesystem       1K-blocks     Used Available Use% Mounted on
/dev/sda3         18244476  5306024  12005028  31% /
tmpfs              2097152   397940   1699212  19% /dev/shm
/dev/sda1           194241    41465    142536  23% /boot
/dev/sdb1         20504628 11566544   7889848  60% /u01
core1:/u01/arch1  20504832 11201024   8255488  58% /u01/arch1
[root@core2 ~]# cd /u01/arch1
[root@core2 arch1]# ll
-rw-r----- 1 oracle asmadmin    5632 Oct 18  2019 2_6_1021623018.dbf
-rw-r----- 1 oracle asmadmin 3063296 Oct 19  2019 2_7_1021623018.dbf
-rw-r----- 1 oracle asmadmin    1024 Oct 19  2019 2_8_1021623018.dbf
-rw-r----- 1 oracle asmadmin  192000 Oct 19  2019 2_9_1021623018.dbf
[root@core2 arch1]# touch test.txt
touch: cannot touch 'test.txt': Permission denied
[root@core2 arch1]# su - oracle
[oracle@core2 ~]$ cd /u01/arch1
[oracle@core2 arch1]$ rm test 
rm: remove write-protected regular empty file 'test'? y   
[oracle@core2 arch1]$ 
```



### 原因阐述

导致此种情况发生的原因是，在Linux层面，用户对该文件不存在写（W）权限。   
在实验中可以看到，当转到Oracle用户的时候便可以删除服务端建立的文件，因为arch1目录的拥有者是Oracle，且拥有者对此目录具有写权限。  

一般情况下，root用户具有一切权限，但是在NFS中确是一个例外，  
NFS共享目录就好像在本地通过网络的方式连接了一个U盘，使得远程的目录能像在本地一样操作， 
当在客户端挂在服务端目录后，root用户的UID和GID都会变成nobody身份，  
即：匿名使用者，从本质上来说客户端root用户被当做成了一个匿名的普通用户来对待。  



###  解决办法

此处如果想在用户权限(非目录权限)方面解决删改客户端目录中内容时，可以通过两种办法解决：  
1. 给other用户添加一个写（X）权限，便可解决。  
chmod o+w /u01/arch1
2. 在exports中指定no_root_squash参数，使客户端root用户具有根目录的完全管理访问权限解决。

##### 注:Linux通过用户的UID来标记用户对此目录是否具有读写权限。





## Linux中对目录和文件不同权限可做的操作：

| 目录属主 | 文件属主 | 目录拥有权限 | 文件拥有权限 | 可否删除文件 | 可否删除目录 |
| :---: | :-----: | :---: | :---: | :---: | :---: |
| root用户 | root用户 | w权限 | 有或无w权限 | 可以 | 不可以 |
| root用户 | root用户 | 无w权限 | 有或无w权限 | 不可以 | 不可以 |
| 普通用户 | root用户 | 有或无w权限 | 有或无w权限 | 可以（root/普通用户） | 可以（root/普通用户） |
| root用户 | 普通用户普通 | 有w权限 | 有或无w权限 | 可以（root/普通用户） | 不可以 |
| root用户 | 普通用户普通 | 无w权限 | 有或无w权限 | 不可以 | 不可以 |



