---
title: Oracle正常安装sqlplus命令无法登录 
date: 2019-10-19
categories: Oracle
tags: Oracle常见问题
---



#

### 版本  
Oracle：11g R2
Linux：Redhat 6

### 错误过程
Oracle重新安装正常结束，但是使用sqlplus命令无法登录，报错如下：
```shell
[oracle@coresu ~]$ sqlplus / as sysdba
-bash: alwrap: command not found
```
一开始开莫名其妙，为什么报错指令找不到！！！  

```shell
[oracle@coresu ~]$ cd $ORACLE_HOME/bin
[oracle@coresu bin]$ ll sqlplus
-rwxr-x--x. 1 oracle oinstall 8991 Oct 16 20:48 sqlplus
[oracle@coresu bin]$ ./sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on 星期日 10月 20 01:53:38 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected.
SYS@prod>
```

发现可以正常登录，突然间恍然大悟，是不是为了给Oracle 11g R2打补丁已解决上下键无法使用的问题，  
配置文件书写有问题。  

```shell 
-bash: alwrap: command not found
```
而且错误上面报是alwrap指令没有找到，仔细翻阅的相关文档发现在参数文件中应该追加的是rlwrap。 

.bash_profile参数文件中追加内容如下：  

```shell

# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

#PATH=$PATH:$HOME/bin

#export PATH

ORACLE_BASE=/u01
ORACLE_HOME=$ORACLE_BASE/oracle
ORACLE_SID=prod
PATH=$ORACLE_HOME/bin:$PATH
export ORACLE_BASE ORACLE_HOME ORACLE_SID
export PATH

alias sqlplus='rlwrap sqlplus'
alias rman='rlwrap rman'

NLS_LANG="simplified chinese"_china.AL32UTF8
export NLS_LANG
export NLS_DATE_FORMAT='YYYY-MM-DD HH24:MI:SS'
export NLS_TIMESTAMP_FORMAT='yyyy-mm-dd HH24:MI:SSXFF'
export NLS_TIMESTAMP_TZ_FORMAT='yyyy-mm-dd HH24:MI:SSXFF  TZR'
```

##### 原因总结:进入到$ORACLE_HOME发现可以./sqlplus可以正常执行，说明sqlplus是正常安装的。在从root用户转到Oracle用户的过程中，会自动运行oracle用户的配置文件.bash_profile，在配置文件中由于我给sqlplus和rman打了补丁，以别名的方式让补丁自动启动，但是由于补丁的名称打错了致使找不到sqlplus软件，从而无法正常登陆。  

##### 错误总结:这个错误体现出我在看系统提示的错误时不仔细，更体现出我个人英语水平的局限性，应在日后的学习中多多加强英语方面的学习，在遇到具体问题是要读明白，想清楚再操作，这样可以节省很多不必要的时间。