---
title:[Oracle]-Centos7下安装RAC弹窗不显示或者显示太小问题解决办法
date:2020-10-22
---



Linux系统版本：红旗redflag 7.6 grid软件版本：11.2.0.3



### 问题现象

在安装grid软件时，从预检查到点击ignore all忽略未通过项开始安装后，界面一直处于卡死状态。



### 问题原因

仔细观察，发现界面中间有一个小弹窗，但是无法显示；

在这个步骤的上一个步骤检查错误时，发现查看响应的报错需要手动拉大才能看见信息；

![image.png](http://cdn.lifemini.cn/dbblog/20201022/2219fb14f3194f9e8f7298dc1476800c.png)

![image.png](http://cdn.lifemini.cn/dbblog/20201022/63e6771018d14686b0730069fdad3108.png)

检查安装日志，发件也仅有一条点击ignore all所记录的warning的记录；

![image.png](http://cdn.lifemini.cn/dbblog/20201022/ec761c261ef940f2bf76c13c97b22580.png)

证明：图形化界面有问题;



### 解决办法

使用如下命令启动图形化界面

```
./runInstaller -jreLoc /etc/alternatives/jre_1.8.0
```

再次运行到此处时，界面正常弹出；
原来是一个忽略警告的选项，不选择yes无法继续进行。

![image.png](http://cdn.lifemini.cn/dbblog/20201022/d9a70892332c4f3bad8224e838956f9f.png)



### 参考文章

- [csdn-sdut菜鸟的文章](https://blog.csdn.net/sdut406/article/details/81463758)