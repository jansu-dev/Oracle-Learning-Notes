---
title:[Oracle]-Centos7.X安装数据库软件出现问题解决
date:2020-10-22
---



Linux系统版本：红旗redflag 7.6 Oracle版本：11.2.0.3



### 问题现象

Linux7.6 64部署 RAC， 在安装 oracle 软件步骤时， 出现错误：Error in invoking target 'agent nmhs' of makefile；

![image.png](http://cdn.lifemini.cn/dbblog/20201022/47e31f51d201436bb47cbf2fdc7754a2.png)



### 解决办法

```
[oracle@XXXX ~]$ cd $ORACLE_HOME/sysman/lib
[oracle@XXXX lib]$ vi ins_emagent.mk
...
(SYSMANBIN)emdctl:(MK_EMAGENT_NMECTL) -lnnz11
...
```



<span style='color:red'>注意：</span>

<span style='color:red'>1. 做好备份；</span>
<span style='color:red'>2. 查找MK_EMAGENT_NMECTL，在后面追加参数 空格 和 -lnnz11 第一个是字母l，后面两个是数字1；</span>

### 参考文章

- [参考文章1-博客园-ritchy的文章](https://www.cnblogs.com/ritchy/p/12055790.html)