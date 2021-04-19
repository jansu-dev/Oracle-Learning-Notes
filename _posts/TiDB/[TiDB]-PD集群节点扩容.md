---
title:[TiDB]-PD集群节点扩容
date:2020-12-27
---



本着觉知此事要躬行的精神，亲自尝试了一次PD集群节点扩容。

### 配置inventory.ini新节点IP信息

```
......
......

[pd_servers]
192.168.1.41
192.168.1.42
192.168.1.43
192.168.1.44

......
......

# node_exporter and blackbox_exporter servers
[monitored_servers]
192.168.1.41
192.168.1.42
192.168.1.43
192.168.1.44

......
......
```



### 中控机操作部署机建用户

执行以下命令，依据输入***部署目标机器\***的 root 用户密码；
本例新增节点IP为192.168.1.44；

```
[tidb@tidb01-41 tidb-ansible]$ vi hosts.ini 
[tidb@tidb01-41 tidb-ansible]$ cat hosts.ini 
[servers]
192.168.1.44


[all:vars]
username = tidb
ntp_server = cn.pool.ntp.org



[tidb@tidb01-41 tidb-ansible]$ ansible-playbook -i hosts.ini create_users.yml -u root -k
SSH password: 

PLAY [all] ***************************************************************************************************************

TASK [create user] *******************************************************************************************************
ok: [192.168.1.44]

TASK [set authorized key] ************************************************************************************************
changed: [192.168.1.44]

TASK [update sudoers file] ***********************************************************************************************
ok: [192.168.1.44]

PLAY RECAP ***************************************************************************************************************
192.168.1.44               : ok=3    changed=1    unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 中控机操作部署机配置ntp服务

<span style=color:red>**注意：生产上应该指向自己的ntp服务器，本次测试采用了公网公用的ntp服务不稳定。**</span>

```
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook -i hosts.ini deploy_ntp.yml -u tidb -b

PLAY [all] ***************************************************************************************************************

TASK [get facts] *********************************************************************************************************
ok: [192.168.1.44]

TASK [RedHat family Linux distribution - make sure ntp, ntpstat have been installed] *************************************
changed: [192.168.1.44] => (item=[u'ntp'])

TASK [RedHat family Linux distribution - make sure ntpdate have been installed] ******************************************
ok: [192.168.1.44] => (item=[u'ntpdate'])

TASK [Debian family Linux distribution - make sure ntp, ntpstat have been installed] *************************************

TASK [Debian family Linux distribution - make sure ntpdate have been installed] ******************************************

TASK [RedHat family Linux distribution - make sure ntpd service has been stopped] ****************************************
ok: [192.168.1.44]

TASK [Debian family Linux distribution - make sure ntp service has been stopped] *****************************************

TASK [Adjust Time | start to adjust time with cn.pool.ntp.org] ***********************************************************
changed: [192.168.1.44]

TASK [RedHat family Linux distribution - make sure ntpd service has been started] ****************************************
changed: [192.168.1.44]

TASK [Debian family Linux distribution - Make sure ntp service has been started] *****************************************

PLAY RECAP ***************************************************************************************************************
192.168.1.44               : ok=6    changed=3    unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 中控及操作部署机设置CPU模式

调整CPU模式，如果同本文出现一样的报错，说明此版本的操作系统不支持CPU模式修改，可直接跳过。

```
[tidb@tidb01-41 tidb-ansible]$ ansible -i hosts.ini all -m shell -a "cpupower frequency-set --governor performance" -u tidb -b
192.168.1.44 | FAILED | rc=237 >>
Setting cpu: 0
Error setting new values. Common errors:
- Do you have proper administration rights? (super-user?)
- Is the governor you requested available and modprobed?
- Trying to set an invalid policy?
- Trying to set a specific frequency, but userspace governor is not available,
   for example because of hardware which cannot be set to a specific frequency
   or because the userspace governor isn't loaded?non-zero return code
```



### 执行bootstrap.yml生成模板文件

```
[tidb@tidb01-41 tidb-ansible]$ 
[tidb@tidb01-41 tidb-ansible]$ vi inventory.ini 
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook bootstrap.yml -l 192.168.1.44

PLAY [initializing deployment target] ************************************************************************************
skipping: no hosts matched

PLAY [check node config] *************************************************************************************************

TASK [pre-ansible : disk space check - fail when disk is full] ***********************************************************
ok: [192.168.1.44]

......
......

PLAY [create ops scripts] ************************************************************************************************
skipping: no hosts matched

PLAY RECAP ***************************************************************************************************************
192.168.1.44               : ok=21   changed=0    unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 执行deploy.yml正式部署新节点

```
[tidb@tidb01-41 tidb-ansible]$ vi inventory.ini 
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook bootstrap.yml -l 192.168.1.44

PLAY [initializing deployment target] ************************************************************************************
skipping: no hosts matched

PLAY [check node config] *************************************************************************************************

TASK [pre-ansible : disk space check - fail when disk is full] ***********************************************************
ok: [192.168.1.44]

......
......

TASK [firewalld : reload firewalld] **************************************************************************************

PLAY RECAP *****************************************************************
```



### 滚动更新普罗米修斯

```
[tidb@tidb01-41 tidb-ansible]$ vi inventory.ini 
[tidb@tidb01-41 tidb-ansible]$ ansible-playbook rolling_update_monitor.yml --tags=prometheus

PLAY [check config locally] ****************************************************************************************************

TASK [check_config_static : Ensure only one monitoring host exists] ************************************************************

......
......

PLAY RECAP *********************************************************************************************************************
192.168.1.41               : ok=3    changed=0    unreachable=0    failed=0   
192.168.1.42               : ok=25   changed=8    unreachable=0    failed=0   
192.168.1.43               : ok=3    changed=0    unreachable=0    failed=0   
localhost                  : ok=7    changed=4    unreachable=0    failed=0   

Congrats! All goes well. :-)
```



### 基于Grafana可视化界面验证

![39368d20da884d3e70484891aa58aae.png](http://cdn.lifemini.cn/dbblog/20201227/b5ceba7b9807484b9f385d382d5ab325.png)