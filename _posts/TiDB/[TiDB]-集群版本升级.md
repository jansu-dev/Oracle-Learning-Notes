---
title:[TiDB]-集群版本升级
date:202012-27
---



#### [TiDB]-集群升级

- [备份原目录](备份原目录)
- [下载新版本](下载新版本)
- [配置inventory.ini](配置inventory.ini)
- [local_prepare下载binary包](local_prepare下载binary包)
- [同步前版本相同配置](同步前版本相同配置)
- [生成模板文件](生成模板文件)
- [开始滚动升级](开始滚动升级)
- [验证升级成功](验证升级成功)





### 备份原目录

```
[tidb@tidb01-41 soft]$ mv tidb-ansible tidb-ansible.v3.0.0
```



### 下载新版本

```
[tidb@tidb01-41 soft]$ git clone -b v3.0.1 https://github.com/pingcap/tidb-ansible.git
Cloning into 'tidb-ansible'...
......
......
PLAY RECAP **************************************************************************************************************************************************
localhost                  : ok=36   changed=18   unreachable=0    failed=0   

Congrats! All goes well. :-)


[tidb@tidb01-41 soft]$ ll |grep tidb-ansible
drwxrwxr-x  14 tidb tidb     4096 Dec 26 11:35 tidb-ansible
drwxrwxr-x. 18 tidb tidb     4096 Dec 23 08:47 tidb-ansible.v3.0.0
```



### 配置inventory.ini

![image.png](http://cdn.lifemini.cn/dbblog/20201226/22246cd1f0044609abeeb084301785b4.png)



### local_prepare下载binary包

```
[tidb@tidb01-41 tidb-ansible]$ pwd
/home/tidb/soft/tidb-ansible
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook local_prepare.yml

PLAY [do local preparation] *********************************************************************************************************************************

TASK [local : Stop if ansible version is too low, make sure that the Ansible version is Ansible 2.4.2 or later, otherwise a compatibility issue occurs.] ****
ok: [localhost] => {
    "changed": false, 
......
......
PLAY RECAP **************************************************************************************************************************************************
localhost                  : ok=36   changed=19   unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 同步前版本相同配置

```
......
......

log:
  # Log level: debug, info, warn, error, fatal.
  level: "debug"

......
......
```



### 生成模板文件

<span style=color:red>注意： 如果你是实验环境，请关闭自检选项，具体步骤参考：[**TiDB-Ansible使用与TiDB部署**](http://www.jannest.com/article/175)</span>
否则会有相关报错出现[192.168.1.43]: Ansible FAILED! => playbook: bootstrap.yml; TASK: check_system_optional : Preflight check - Check TiDB server's RAM; message: {"changed": false, "msg": "This machine does not have sufficient RAM to run TiDB, at least 16000 MB."}

```
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook bootstrap.yml 

PLAY [initializing deployment target] 
......
......
PLAY RECAP **************************************************************************************************************************************************
192.168.1.41               : ok=21   changed=0    unreachable=0    failed=0   
192.168.1.42               : ok=21   changed=0    unreachable=0    failed=0   
192.168.1.43               : ok=21   changed=0    unreachable=0    failed=0   
localhost                  : ok=9    changed=6    unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 开始滚动升级

```
[tidb@tidb01-41 tidb-ansible]$ pwd
/home/tidb/soft/tidb-ansible
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook rolling_update.yml 

PLAY [check config locally] *********************************************************************************************************************************

......
......

TASK [wait until the TiDB status page is available when enable_tls] *****************************************************************************************

PLAY RECAP **************************************************************************************************************************************************
192.168.1.41               : ok=120  changed=18   unreachable=0    failed=0   
192.168.1.42               : ok=94   changed=12   unreachable=0    failed=0   
192.168.1.43               : ok=119  changed=18   unreachable=0    failed=0   
localhost                  : ok=7    changed=4    unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 验证升级成功

```
[tidb@tidb01-41 tidb-ansible]$ mysql -u root -h 192.168.1.41 -P 4000
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.25-TiDB-v3.0.1 MySQL Community Server (Apache License 2.0)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]>
```