---
title:[Oracle]-DBCA建库时出现Unable copy the file...问题解决办法
date:2020-10-22
---



Linux系统版本：红旗redflag 7.6 Oracle版本：11.2.0.3



### 问题现象

在dbca建库的最后一步，点击Finish之后，出现Unable copy the file...错误；

![image.png](http://cdn.lifemini.cn/dbblog/20201022/6e090fd2f6ba44a3935998858463f51b.png)



### 解决办法

首先，检查结果显示文件存在且权限正常；

```
[root@XXX tmp]# ls -ltr /etc/oratab 
-rw-rw-r-- 1 oracle oinstall 904 Apr 22 10:14 /etc/oratab
```

根据图片中的报错来看，是操作：从另一个节点的“/etc/oratab”文件读取信息到本节点的“/tmp/oratab.HOSTNAME”文件无法执行；

此时，手动将另一个节点的oratab信息写入“/tmp/oratab.HOSTNAME”中；

然后，选择 ignore 选项，忽略这一错误。

最后，经过手动处理后dbca会自动将相应信息补全全到两节点的/etc/oratab文件中，不影响安装结果。

![image.png](http://cdn.lifemini.cn/dbblog/20201022/62f14246341b4f9cb366444e5a88294f.png)





### 参考文章

- [参考文章1-csdn-weixin_34149796的文章](https://blog.csdn.net/weixin_34149796/article/details/90622004)

