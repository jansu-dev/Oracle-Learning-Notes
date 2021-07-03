---
title:[Oracle]--11g RAC集群及数据库软件干净删除
date:2020-10-22
---





#### 11g RAC集群及数据库软件干净删除

- ###### [删除软件](删除软件)

- ###### [清理目录](清理目录)

- ###### [清理ASM信息](清理ASM信息)





### 删除软件

##### 卸载Oracle 11g rac database软件

```
[oracle@rac01 ~]$ cd /usr/app/product/oracle/11.2.0/deinstall/  

[oracle@rac01 deinstall]$ ./deinstall
```

##### 卸载Oracle 11g rac grid软件

```
[grid@rac01 ~]$ cd /usr/app/product/grid/11.2.0/deinstall/  

[grid@rac01 deinstall]$ ./deinstall
```



### 清理目录

```
[root@rac_1 grid]# rm -rf /etc/ora*
[root@rac_1 grid]# rm -rf /oracle/app/grid/*
[root@rac_1 grid]# rm -rf /usr/local/bin/dbhome 
[root@rac_1 grid]# rm -rf /usr/local/bin/oraenv 
[root@rac_1 grid]# rm -rf /usr/local/bin/coraenv 
[root@rac_1 grid]# rm -rf /oracle/app/oraInventory
```



### 清理ASM信息

<span style='color:red'>注意：案例语句为多路径共享盘</span>

```
dd if=/dev/zero of=/dev/mapper/mpath0 bs=10M count=10
dd if=/dev/zero of=/dev/mapper/mpath1 bs=10M count=10
dd if=/dev/zero of=/dev/mapper/mpath2 bs=10M count=10
```