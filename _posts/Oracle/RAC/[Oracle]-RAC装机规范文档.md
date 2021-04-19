---
title:[Oracle]-RAC装机规范文档
date:2020-11-10
---



##### [Oracle\]-RAC装机规范文档:

- [装GRID](装GRID)
- [安装数据库软件](安装数据库软件)
- [配置ASM磁盘](配置ASM磁盘)
- [装库](装库)
- [参数修改](参数修改)



### 装GRID

##### 系统要求：

库：tmsreport
节点1：jandb-db-05 sid: tmsreport1 bond0：10.3.50.245
节点2：jandb-db-06 sid: tmsreport2 bond0：10.3.50.246

操作系统版本：2.6.18-194.1.AXS3

安装操作系统的时候，需要的rpm包要全部装上，防火墙关掉；

1. 修改解析配置文件
   Localhost这个回环地址要留着

```
[root@jandb-db-05 ~]# vim /etc/hosts
#LOCAL
127.0.0.1                localhost.localdomain   localhost
::1                     localhost6.localdomain6 localhost6
# Pubilc Host Names
10.3.50.245       jandb-db-05
10.3.50.246       jandb-db-06
# Private Host Names
192.168.1.5       jandb-db-05-priv
192.168.1.6       jandb-db-06-priv
# Virtual Host Names
10.3.50.247       jandb-db-05-vip
10.3.50.248       jandb-db-06-vip
# Scan ip
10.3.50.250       jandb-report-scan

把DNS解析的配置文件重名命，这里解析只走hosts
[root@jandb-db-05 /]#  mv /etc/resolv.conf /etc/resolv.conf_bak
```

2. 修改虚拟内存
   在配置文件的挂载大小，添加红色部分字体

```
[root@jandb-db-05 ~]# vim /etc/fstab
/dev/vg00/lv_root           /                       ext3    defaults                 1 1
/dev/vg00/lv_home         /home                   ext3    defaults                 1 2
/dev/vg00/lv_tmp          /tmp                    ext3    defaults                 1 2
LABEL=/boot             /boot                   ext3    defaults                 1 2
tmpfs                    /dev/shm                tmpfs   defaults,size=42G         0 0
devpts                   /dev/pts                devpts  gid=5,mode=620           0 0
sysfs                     /sys                    sysfs    defaults                 0 0
proc                     /proc                   proc    defaults                 0 0
/dev/vg00/lv_swap         swap                   swap    defaults                 0 0
[root@jandb-db-05 ~]#  mount -o remount /dev/shm/
[root@jandb-db-05 ~]#  df -lh
```

3. 修改内核参数
   首先注销掉配置文件最后6行内容，然后添加下面8行内容。

```
[root@jandb-db-05 ~]# vim /etc/sysctl.conf
##################11g#############################
net.core.rmem_max=4194304
net.core.wmem_max=1048576
net.core.rmem_default=262144
net.core.wmem_default=262144
net.ipv4.ip_local_port_range =9000 65500
fs.file-max = 6815744
fs.aio-max-nr =1048576
kernel.shmmni =4096
-------超多cpu 集群等待高的问题比如10.2.195.130/131时效的数据量
net.ipv4.ipfrag_low_thresh = 943718400  ---900M
net.ipv4.ipfrag_high_thresh = 1073741824  ---1024M
net.ipv4.ipfrag_time = 90
net.core.rmem_max = 16777216----16m
net.core.wmem_max = 16777216----16m

[root@jandb-db-05 ~]# sysctl -p
```

4. 配置时间服务器

```
[root@jandb-db-05 ~]# vim /etc/sysconfig/ntpd
# Drop root to id 'ntp:ntp' by default.
OPTIONS="-x -u ntp:ntp -p /var/run/ntpd.pid"
# Set to 'yes' to sync hw clock after successful ntpdate
SYNC_HWCLOCK=yes
# Additional options for ntpdate
NTPDATE_OPTIONS=""
[root@jandb-db-05 ~]# vim /etc/ntp.conf
server  10.3.36.155
server  10.3.36.130
[root@jandb-db-05 ~]# vim /etc/ntp/step-tickers
10.3.36.155
10.3.36.130
[root@jandb-db-05 ~]# ntpdate 10.3.36.155
[root@jandb-db-05 ~]# /etc/init.d/ntpd start
[root@jandb-db-05 ~]# chkconfig ntpd on
```

5. 修改最大限制

```
[root@jandb-db-05 ~]# vim /etc/profile
if [ $USER = "oracle" ]||[ $USER = "grid" ]; then
        if [ $SHELL = "/bin/ksh" ]; then
                ulimit -p 16384
                ulimit -n 65536
        else
                ulimit -u 16384 -n 65536
        fi
                umask 022
fi
[root@jandb-db-05 ~]# vim /etc/security/limits.conf
grid            soft    nproc   2047
grid            hard    nproc   16384
grid            soft    nofile  1024
grid            hard    nofile  65536
oracle          soft    nproc   2047
oracle          hard    nproc   16384
oracle          soft    nofile  1024
oracle          hard    nofile  65536
[root@jandb-db-05 ~]# vim /etc/pam.d/login
session    required     pam_limits.so
```

6. 创建组和用户

```
groupadd -g 1000 oinstall
groupadd -g 1001 dba
groupadd -g 1002 oper
groupadd -g 1003 asmadmin
groupadd -g 1004 asmdba
groupadd -g 1005 asmoper
useradd -u 2000 -g oinstall -G asmadmin,asmdba,asmoper,dba -d /home/grid grid
useradd -u 2001 -g oinstall -G asmdba,asmadmin,dba,oper -d /home/oracle oracle

passwd grid
passwd oracle
```

7. 创建目录

```
mkdir -p /oracle/app/grid
mkdir -p /oracle/app/11.2.0/grid
chown -R grid:oinstall /oracle
mkdir -p /oracle/app/oracle
chown -R oracle:oinstall /oracle/app/oracle
chmod -R 775 /oracle
```

8. 设置环境变量
   两个bash文件都没有设置home的环境变量，等全部装完后再补上

   Grid用户：添加如下内容

```
[root@jandb-db-05 ~]# vim /home/grid/.bash_profile
export ORACLE_SID=+ASM1[2]
export ORACLE_BASE=/oracle/app/grid
export ORACLE_OWNER=grid
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib:/lib:/usr/lib:/usr/local/lib
export NLS_LANG="simplified chinese"_china.zhs16gbk
export PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin
unset USERNAME
umask 022
```

​	Oracle用户：添加如下内容

```
[root@jandb-db-05 ~]# vim /home/oracle/.bash_profile
export ORACLE_UNQNAME=tmsreport
export ORACLE_SID=tmsrpt1[2]
export ORACLE_BASE=/oracle/app/oracle
export ORACLE_OWNER=oracle
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib:/lib:/usr/lib:/usr/local/lib
export NLS_LANG="simplified chinese"_china.zhs16gbk
export PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin
unset USERNAME
alias sqlplus="rlwrap sqlplus"
umask 022
[root@jandb-db-05 ~]# source /home/grid/.bash_profile
[root@jandb-db-05 ~]# source /home/oracle/.bash_profile
```

9. 配置裸设备
   现在只装grid，用前面三块5G的共享盘，现在还没分区，先划分区(以后做的时候应该把所有要用到的磁盘一次性划分完加入裸设备里面)
   因为是共享存储，只在一个节点上划分区，划完后在第二个节点刷下分区表就可以了

```
[root@jandb-db-05 ~]# sfdisk -s
/dev/cciss/c0d0: 292935982
/dev/sda:   5242880
/dev/sdb:   5242880
/dev/sdc:   5242880
/dev/sdd: 419430400
/dev/sde: 419430400
/dev/sdf: 367001600
/dev/sdg: 367001600
/dev/sdh: 361758720
/dev/sdi: 146800640
total: 2390087982 blocks



[oracle@nx-app-83 scripts]$ cat /etc/udev/rules.d/60-raw.rules 
# Enter raw device bindings here.
#
# An example would be:
#   ACTION=="add", KERNEL=="sda", RUN+="/bin/raw /dev/raw/raw1 %N"
# to bind /dev/raw/raw1 to /dev/sda, or
#   ACTION=="add", ENV{MAJOR}=="8", ENV{MINOR}=="1", RUN+="/bin/raw /dev/raw/raw2 %M %m"
# to bind /dev/raw/raw2 to the device with major 8, minor 1.

 ACTION=="add", KERNEL=="mpath0", RUN+="/bin/raw /dev/raw/raw1 %N"
 ACTION=="add", KERNEL=="mpath1", RUN+="/bin/raw /dev/raw/raw2 %N"
 ACTION=="add", KERNEL=="mpath2", RUN+="/bin/raw /dev/raw/raw3 %N"
 ACTION=="add", KERNEL=="mpath3", RUN+="/bin/raw /dev/raw/raw4 %N"
 ACTION=="add", KERNEL=="mpath4", RUN+="/bin/raw /dev/raw/raw5 %N"
 ACTION=="add", KERNEL=="mpath5", RUN+="/bin/raw /dev/raw/raw6 %N"
 ACTION=="add", KERNEL=="mpath6", RUN+="/bin/raw /dev/raw/raw7 %N"
 ACTION=="add", KERNEL=="mpath7", RUN+="/bin/raw /dev/raw/raw8 %N"
 ACTION=="add", KERNEL=="mpath8", RUN+="/bin/raw /dev/raw/raw9 %N"

[oracle@nx-app-83 scripts]$ cat /etc/rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local
/bin/raw /dev/raw/raw1 /dev/mapper/mpath0
/bin/raw /dev/raw/raw2 /dev/mapper/mpath1
/bin/raw /dev/raw/raw3 /dev/mapper/mpath2
/bin/raw /dev/raw/raw4 /dev/mapper/mpath3
/bin/raw /dev/raw/raw5 /dev/mapper/mpath4
/bin/raw /dev/raw/raw6 /dev/mapper/mpath5
/bin/raw /dev/raw/raw7 /dev/mapper/mpath6
/bin/raw /dev/raw/raw8 /dev/mapper/mpath7
/bin/raw /dev/raw/raw9 /dev/mapper/mpath8
chown grid:asmadmin /dev/raw/raw*
chmod 660 /dev/raw/raw*
```

​	配置设备参数文件，重启服务器

```
[root@jandb-db-05 ~]# vim /etc/sysconfig/rawdevices
/dev/raw/raw1 /dev/sda1
/dev/raw/raw2 /dev/sdb1
/dev/raw/raw3 /dev/sdc1

[root@jandb-db-05 ~]# /etc/init.d/rawdevices restart
Assigning devices:
           /dev/raw/raw1  -->   /dev/sda1  /dev/raw/raw1:  bound to major 8, minor 1
           /dev/raw/raw2  -->   /dev/sdb1  /dev/raw/raw2:  bound to major 8, minor 17
           /dev/raw/raw3  -->   /dev/sdc1  /dev/raw/raw3:  bound to major 8, minor 33
done
```

​	运行下面命令修改权限并写入开机启动

```
[root@jandb-db-05 ~]# chown grid:asmadmin /dev/raw/raw*
[root@jandb-db-05 ~]# chmod 660 /dev/raw/raw*

[root@jandb-db-05 ~]# vim /etc/rc.local
chown grid:asmadmin /dev/raw/raw*
chmod 660 /dev/raw/raw*

[root@jandb-db-05 ~]# ll /dev/raw
总计 0
crw-rw---- 1 grid asmadmin 162, 1 01-04 14:23 raw1
crw-rw---- 1 grid asmadmin 162, 2 01-04 14:23 raw2
crw-rw---- 1 grid asmadmin 162, 3 01-04 14:23 raw3
```

10. 配置无密码访问
    切换用户到grid下

    节点1:

```
mkdir ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa
ssh-keygen -t dsa
```

​	节点2：

```
mkdir ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa
ssh-keygen -t dsa
```

​	节点1：

```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
ssh jandb-db-06 cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
ssh jandb-db-06 cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys jandb-db-06:~/.ssh/authorized_keys
```

​	配置完成后两个节点再运行如下命令

```
[grid@jandb-db-05 ~]$ ssh jandb-db-06-priv date
[grid@jandb-db-05 ~]$ ssh jandb-db-05-priv date
[grid@jandb-db-05 ~]$ ssh jandb-db-06 date
[grid@jandb-db-05 ~]$ ssh jandb-db-05 date
```

11. 图像界面安装grid

```
grid的包是 ：p10404530_112030_Linux-x86-64_3of7.zip
[root@jandb-db-05 /]# unzip p10404530_112030_Linux-x86-64_3of7.zip
[root@jandb-db-05 /]# cd /grid/rpm/
[root@jandb-db-05 rpm]# rpm -ivh cvuqdisk-1.0.9-1.rpm
[root@jandb-db-05 /]# chown grid:oinstall grid/* -R

[grid@jandb-db-05 grid]$ 
./runcluvfy.sh stage -pre crsinst -n jandb-db-05,jandb-db-06 -fixup -verbose

[grid@jandb-db-05 grid]$ export DISPLAY=10.3.7.234:0.0
[grid@jandb-db-05 grid]$ ./runInstaller
```

​	在执行root.sh脚本之前先做如下改动：
​	（/oracle/app/11.2.0/grid是grid的home目录，环境变量里面没有写出来，是操作步骤第13步的那个software loc的路径）
​	修改1：在这个文件的894行中间添加红色字体

```
[grid@jandb-db-05 ~]$ vim /oracle/app/11.2.0/grid/lib/osds_acfsroot.pm
 if (!(($minor eq "32-100") || ($minor eq "32-71") ||
          ($variation eq 'el5') || ($variation eq 'AXS3') || ($variation eq 'el5PAE') ||
          ($variation eq 'el5xen')))
    {
      lib_error_print(9384, "Invalid OS kernel variation '%s'.", $variation);
      lib_error_print(9135, "%s installation aborted.", $COMMAND);
      exit USM_NOT_SUPPORTED;
    }
```

​	修改2：到下面这个路径下拷贝生成这些目录

```
[grid@jandb-db-05 ~]$ cd /oracle/app/11.2.0/grid/install/usm/EL5/x86_64/2.6.18-8/ 
cp -a 2.6.18-8.el5-x86_64 `uname -r`-x86_64
cp -a 2.6.18-8.el5-x86_64 `uname -r`
cp -a 2.6.18-8.el5-x86_64 `uname -r`-86_64
cp -a 2.6.18-8.el5-x86_64  2.6.18-8.AXS3-x86_64
cp -a 2.6.18-8.el5-x86_64  2.6.18-8.AXS3
cp -a 2.6.18-8.el5-x86_64  2.6.18-8.AXS3-86_64

Root.sh脚本执行成功后最后一行是
Configure Oracle Grid Infrastructure for a Cluster ... Succeeded
```

​	安装完成后检查集群资源，只有gsd是offline这是正常的，整个集群安装成功

```
[grid@jandb-db-05 /]$ cd /oracle/app/11.2.0/grid/bin/
[grid@jandb-db-05 bin]$ ./crs_stat -t
名称                 类型           目标     状态      主机


ora.DATACS.dg     ora....up.type    ONLINE    ONLINE    effe...b-05
ora....ER.lsnr        ora....er.type    ONLINE    ONLINE    effe...b-05
ora....N1.lsnr        ora....er.type    ONLINE    ONLINE    effe...b-05
ora.asm            ora.asm.type    ONLINE    ONLINE    effe...b-05
ora.cvu            ora.cvu.type     ONLINE    ONLINE    effe...b-05
ora....SM1.asm      application      ONLINE    ONLINE    effe...b-05
ora....05.lsnr        application      ONLINE    ONLINE    effe...b-05
ora....-05.gsd        application      OFFLINE   OFFLINE
ora....-05.ons        application      ONLINE    ONLINE    effe...b-05
ora....-05.vip        ora....t1.type     ONLINE    ONLINE    effe...b-05
ora....SM2.asm      application      ONLINE    ONLINE    effe...b-06
ora....06.lsnr        application      ONLINE    ONLINE    effe...b-06
ora....-06.gsd        application      OFFLINE   OFFLINE
ora....-06.ons        application      ONLINE    ONLINE    effe...b-06
ora....-06.vip        ora....t1.type     ONLINE    ONLINE    effe...b-06
ora.gsd             ora.gsd.type     OFFLINE   OFFLINE
ora....network       ora....rk.type     ONLINE    ONLINE    effe...b-05
ora.oc4j            ora.oc4j.type     ONLINE    ONLINE    effe...b-05
ora.ons             ora.ons.type      ONLINE    ONLINE    effe...b-05
ora....ry.acfs         ora....fs.type      ONLINE    ONLINE    effe...b-05
ora.scan1.vip         ora....ip.type     ONLINE    ONLINE    effe...b-05
```

​	到这里整个grid安装成功



### 安装数据库软件

配置oracle用户的无密码访问
切换用户到oracle下
节点1:

```
mkdir ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa
ssh-keygen -t dsa
```

节点2:

```
mkdir ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa
ssh-keygen -t dsa
```

节点1：

```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
ssh effectiveness-db-06 cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
ssh effectiveness-db-06 cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys effectiveness-db-06:~/.ssh/authorized_keys
```

配置完成后两个节点再运行如下命令

```
[grid@effectiveness-db-05 ~]$ ssh effectiveness-db-06-priv date
[grid@effectiveness-db-05 ~]$ ssh effectiveness-db-05-priv date
[grid@effectiveness-db-05 ~]$ ssh effectiveness-db-06 date
[grid@effectiveness-db-05 ~]$ ssh effectiveness-db-05 date




[root@effectiveness-db-05 /]# unzip p10404530_112030_Linux-x86-64_1of7.zip
[root@effectiveness-db-05 /]# unzip p10404530_112030_Linux-x86-64_2of7.zip
[root@effectiveness-db-05 /]# chown oracle:oinstall database/ -R
[root@effectiveness-db-05 /]# su - oracle
[oracle@effectiveness-db-05 database]$ export DISPLAY=10.3.7.234:0.0
[oracle@effectiveness-db-05 database]$ ./runInstaller
```

根据提示在root用户下面依次运行上面脚本

```
[oracle@effectiveness-db-05 ~]$ sqlplus  / as sysdba
SQL*Plus: Release 11.2.0.3.0 Production on  
查看版本已经为11.2.0.3.0，此版本无需再升级
```



### 配置ASM磁盘

```
[root@effectiveness-db-05 ~]# su - grid
[grid@effectiveness-db-05 ~]$ export DISPLAY=10.3.7.234:0.0
[grid@effectiveness-db-05 ~]$ asmca
```

点击下面的create按钮创建一个新的asm组

填上组名，选择外部冗余，选择磁盘，然后点击OK

依次重复上面步骤，建立完成后点击exit退出即可



### 装库

```
[root@effectiveness-db-05 ~]# su - oracle
[oracle@effectiveness-db-05 ~]$ export DISPLAY=10.3.7.234:0.0
[oracle@effectiveness-db-05 ~]$ cd /oracle/app/oracle/product/11.2.0/dbhome_1/bin/
[oracle@effectiveness-db-05 bin]$ ./dbca
```

这里全局名称和SID前缀名称不一样的原因是SID的长度有限制，下次取名字的时候一定注意这个问题

内存参数先这样设置，把数据库装完之后根据需求再做相应调整

字符集和国家字符一定要注意，设置错误了只能删库重建

在两个节点上面分别把oracle和grid用户的家目录写入bash文件

```
[grid@effectiveness-db-05 ~]$ vim /home/grid/.bash_profile
export ORACLE_HOME=/oracle/app/11.2.0/grid
[oracle@effectiveness-db-05 ~]$ vim /home/oracle/.bash_profile
export ORACLE_HOME=/oracle/app/oracle/product/11.2.0/dbhome_1
```



### 参数修改

- 配置初始化参数

```
修改processes 参数为3500，sessions的值也会自动更改
修改open_cursors参数为800
```

- 配置控制文件

```
SQL> select name from v$controlfile;
NAME
------------------------
+DATA1/ic/control01.ctl
+DATA1/ic/control02.ctl

两个节点运行下面命令
SQL> alter system set control_files='+DATA1/ic/control01.ctl',
'+DATA1/ic/control02.ctl','+DATA2/ic/control03.ctl','+DATA2/ic/control04.ctl'
scope=spfile;

分别关闭两个数据库，再将两个库打开到nomount状态
在一个节点上进入rman执行下面命令
Rman>restore controlfile to '+DATA2/ic/control03.ctl' from '+DATA1/ic/control01.ctl';
Rman>restore controlfile to '+DATA2/ic/control04.ctl' from '+DATA1/ic/control01.ctl';
然后打开数据库就可以了
```

- 开启配置归档

```
两个节点分别运行下面命令
SQL> alter system set log_archive_dest_1='location=+DATA10';
然后将数据库都启动到Mount状态，在两个节点运行下面命令
SQL>alter database archivelog
两个节点都要运行完后再去打开数据库
```

- 配置redo

```
select * from v$log
select * from v$logfile
alter database drop logfile group n;
alter system switch logfile
alter system checkpoint;

alter database add logfile thread 1 group 1 ( '+DATA1/ic/redo_01.log') SIZE 1G;
alter database add logfile thread 2 group 2 ( '+DATA1/ic/redo_02.log’) SIZE 1G;

alter database add logfile thread 1 group 3 ( '+DATA2/ic/redo03.log') SIZE 1G;
alter database add logfile thread 2 group 4 ( '+DATA2/ic/redo04.log') SIZE 1G;

alter database add logfile thread 1 group 5 ( '+DATA1/ic/redo05.log') SIZE 1G;
alter database add logfile thread 2 group 6 ( '+DATA1/ic/redo06.log') SIZE 1G;

alter database add logfile thread 1 group 7 ( '+DATA2/ic/redo07.log') SIZE 1G;
alter database add logfile thread 2 group 8 ( '+DATA2/ic/redo08.log') SIZE 1G;
```

- 配置undo:

```
create undo tablespace undo11 datafile '+DATA1' size 30g autoextend off;
alter system set undo_tablespace='UNDO11' SCOPE=BOTH;
```