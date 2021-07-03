---
title:[PG]--cscope+gdb调式步骤详解
date:2020-07-19

---

##### cscope+gdb调式步骤详解。

- [安装gdb和cscope](安装gdb和cscope)
- [cscope使用](cscope使用)
- [gdb使用](gdb使用)
- [参考文章](参考文章)





### 安装gdb和cscope

```
yum install -y gdb
yum install -y cscope
```



### cscope使用

1. 进入所要调试的项目目录的根目录，创建索引链接库，
   系统会自动产生cscope.out、cscope.in.out和cscope.po.out三个文件。

-R：递归解析所有的子目录。
-q：通过倒排索引加速符号的查找过程。
-b：构建交叉引用(cross-reference)文件（链接库）。

```
cscope -Rbq
```

1. 随便开启一个文件，
   本例采用Postgresql的makefile，
   并增加一个新的cscope数据库。

```
vi Makefile

:cs add cscope.out
```

1. 定位查找所需信息

```
:cs find g XXX
```



### gdb使用

### 注意事项

在安装PostgreSQL时，要带上enable_debug参数，否则就会如图行色箭头一样，函数的栈数据无法捕获。
![1.png](http://cdn.lifemini.cn/dbblog/20200719/7817b735836644c79095e8456e426ea1.png)



### 参考文章

- [csdn-skyzhd的文章](https://blog.csdn.net/zhang2531/article/details/51538658)