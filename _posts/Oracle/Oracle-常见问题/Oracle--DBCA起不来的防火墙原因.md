---
title: 完美解决xhost +报错： unable to open display ""
date: 2019-9-13
categories: Oracle
tags: Oracle常见问题
---


# 


X manager在弹出安装界面的时候出错  
出现界面就是蹦不出来的情况


root用户:执行xhost +
再切换到oracle用户:执行export DISPLAY=:0.0
出现乱码执行export LANG=US_en


DISPLAY变量是用来设置将图形显示到何处.
DISPLAY自动设置为DISPLAY=:0.0表示显式到本地监视器  
如果没有设置,系统是不允许程序启动的。
在执行xhost +命令（使得所有客户都可以访问）


先切换到root用户，执行xhost +（）
正常返回：access control disabled,clients can connect from any host
切回oracle用户，执行：export DISPLAY=192.168.1.2：0.0便可运行。



## 如果仍无法运行  
可能是图形界面端（通常为Windows端）的防火墙未关闭导致  
可关闭防火墙开启图形化界面。