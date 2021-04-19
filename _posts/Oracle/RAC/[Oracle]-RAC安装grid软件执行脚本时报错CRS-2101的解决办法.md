---
title:[Oracle]-RAC安装grid软件执行脚本时报错CRS-2101的解决办法
date:2020-10-22
---



Linux系统版本：红旗redflag 7.6 grid软件版本：11.2.0.3



### 问题现象

在安装grid软件，执行到运行/oracle/app/11.2.0/grid/root.sh脚本时出现如下报错，并卡住不动；
关键报错：[client(14360)]CRS-2101:The OLR was formatted using version 3.

![image.png](http://cdn.lifemini.cn/dbblog/20201022/8322445e7c614ac9b157b66f12e50100.png)



### 解决办法

出现如上问题时，ctrl+c结束脚本运行；
重新运行该脚本；

```
[root@XXXX ~]# /oracle/app/11.2.0/grid/root.sh
```

另开一个窗口,在重新运行该脚本的同时向npohasd文件写入null值；

```
[root@XXXX ~]# cd /var/tmp/.oracle/
[root@XXXX ~]# dd if=npohasd of=/dev/null bs=1024 count=1
```

可以看到问题成功解决；
![image.png](http://cdn.lifemini.cn/dbblog/20201022/69c4b5e786c3449f8434547288bc2053.png)

安装完grid软件可能存在reboot后,只有进程ohasd.bin的状况;
为防止该种情况发生，建议在安装完grid软件后执行下面操作。

解决方法：

```
[root@XXXX ~]# cd /var/tmp/.oracle/
[root@XXXX ~]# rm -rf  npohasd
[root@XXXX ~]# touch  npohasd
[root@XXXX ~]# chmod 755  npohasd
```



### 参考文章

- [参考文章1-iptub-adamjunior的文章](http://blog.itpub.net/69959726/viewspace-2686471/)