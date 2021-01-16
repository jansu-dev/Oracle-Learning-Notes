### 文章概览  
- [问题起因](#问题起因)
- [解决问题](#解决问题)
- [测试代码](#测试代码)
- [参考文章](#参考文章)



### 问题起因

在运行shell脚本时，会在当前进程的基础上创建一个子进程，也就造成了环境变量是单项传递的，子进程能接受和在子进程环境变量中修改父进程的环境变量，但是当脚本运行结束后所有修改全部失效，因为子进程已经结束。   
如果直接在SHELL脚本中使用export参数到环境变量，就会出现上述问题，即：使用env | grep抓不到我们在脚本中导出的参数。



### 解决问题  

使用“.”命令可以使得父子进程处于同一级别。  
格式：. ./XXX.sh (注意两个点之间有一个空格，前一个点是命令，后面的点及后面语句表示脚本路径)



### 测试代码  

##### 部分SHELL代码
```
......
......
echo "表空间剩余容量是："$v_tbspace_free
if [[ `echo "$v_tbspace_free < $v_free_tbspace_threshold" | bc` -eq 1 ]];then
    v_tbspace_flag="1"
else
    v_tbspace_flag="0"
fi
export v_tbspace_flag
env | grep v_tbspace_flag
......
......
```
##### 测试结果
```
[oracle@appserver01 ztest]$ . ./test.sh
表空间剩余容量是：55.27
v_tbspace_flag=1

[oracle@appserver01 ztest]$ env |grep v_tbspace
v_tbspace_flag=1
```



### 参考文章  

[参考文章—shell子进程修改父进程的环境变量值](https://blog.csdn.net/xymyeah/article/details/88249387)