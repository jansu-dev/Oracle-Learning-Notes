---
title:[Oracle]-Centos7.6安装RAC配置多路径裸设备时出现无法识别情况问题解决
date:2020-10-20
---



Linux系统版本：红旗redflag 7.6(等同于Centos7.X)



### 问题现象

使用/etc/rc.local配置多路几个裸设备映射的时候，无法生效；
配置文件：

```
[root@XXXX ~]# vi /etc/rc.local

[root@XXXX ~]# cat /etc/rc.local
#!/bin/bash

touch /var/lock/subsys/local
/bin/raw /dev/raw/raw1 /dev/mapper/mpathocr1
/bin/raw /dev/raw/raw2 /dev/mapper/mpathocr2
/bin/raw /dev/raw/raw3 /dev/mapper/mpathocr3
/bin/raw /dev/raw/raw4 /dev/mapper/mpathdata1
/bin/raw /dev/raw/raw5 /dev/mapper/mpathdata2
/bin/raw /dev/raw/raw6 /dev/mapper/mpathdata3
/bin/raw /dev/raw/raw7 /dev/mapper/mpathdata4
/bin/raw /dev/raw/raw8 /dev/mapper/mpathdata5
/bin/raw /dev/raw/raw9 /dev/mapper/mpatharc
chown grid:asmadmin /dev/raw/raw*
chmod 660 /dev/raw/raw*

[root@XXXX ~]# reboot
```

结果现象：

```
[root@XXXX ~]# ll /dev/raw/
总用量 0
```

centos7中需要开启rc.local服务

```
[root@XXXX ~]# chmod +x /etc/rc.d/rc.local

[root@XXXX ~]# systemctl enable rc-local

[root@XXXX ~]# systemctl start rc-local

[root@XXXX ~]# systemctl is-active rc-local
```

再次观察结果现象：

```
[root@XXXX ~]# ll /dev/raw/
总用量 0
crw-rw----. 1 grid asmadmin 162, 1 10月 23 01:35 raw1
crw-rw----. 1 grid asmadmin 162, 2 10月 23 01:35 raw2
crw-rw----. 1 grid asmadmin 162, 3 10月 23 01:35 raw3
crw-rw----. 1 grid asmadmin 162, 4 10月 23 01:35 raw4
crw-rw----. 1 grid asmadmin 162, 5 10月 23 01:35 raw5
crw-rw----. 1 grid asmadmin 162, 6 10月 23 01:35 raw6
crw-rw----. 1 grid asmadmin 162, 7 10月 23 01:35 raw7
crw-rw----. 1 grid asmadmin 162, 8 10月 23 01:35 raw8
crw-rw----. 1 root          162, 9 10月 23 01:35 raw9
crw-rw----. 1 root          162, 0 10月 23 01:35 rawctl
```

而且两个节点相互不对应；
多次重启之后，后面随机出现权限配置不上的现象。



### 解决办法

在/etc/rc.local之前，追加sleep时间。

![image.png](http://cdn.lifemini.cn/dbblog/20201022/ae626f8ba2764529a39a960a82b2a554.png)



### 问题原因

怀疑：执行权限的时候,有的裸设备还没出来,查看系统日志里也没有报错。就是裸设备的初始化可能需要一定时间，执行权限时/dev/raw/下还有没有相应权限。