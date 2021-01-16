
#### top命令查看
```
    top - 16:15:05 up 6 days,  6:25,  2 users,  load average: 1.45, 1.77, 2.14
    Tasks: 29 total, 1 running, 28 sleeping, 0 stopped, 0 zombie
    Cpu(s): 0.3% us, 1.0% sy, 0.0% ni, 98.7% id, 0.0% wa, 0.0% hi, 0.0% si
    Mem:   4037872k total,  4003648k used,    34224k free,     5512k buffers
    Swap:  7164948k total,   629192k used,  6535756k free,  3511184k cached
```
查看12.6% wa,IO等待所占用的CPU时间的百分比,高过30%时IO压力高   
具体的解释如下：   
Tasks:  

- 29 total 进程总数   
- 1 running 正在运行的进程数      
- 28 sleeping 睡眠的进程数     
- 0  stopped 停止的进程数    
- 0 zombie 僵尸进程数   

Cpu(s):   

- 0.3% us 用户空间占用CPU百分比   
- 1.0% sy 内核空间占用CPU百分比   
- 0.0% ni 用户进程空间内改变过优先级的进程占用CPU百分比   
- 98.7% id 空闲CPU百分比   
- 0.0% wa 等待输入输出的CPU时间百分比    
- 0.0% hi   
- 0.0% si  


#### 用iostat -x 1 10

```
　　avg-cpu:  %user   %nice    %sys     %iowait   %idle

　　0.00       0.00     0.25    33.46    66.29

　　Device:    rrqm/s  wrqm/s   r/s    w/s     rsec/s   wsec/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util

　　sda          0.00    0.00      0.00   0.00    0.00    0.00         0.00     0.00     0.00           0.00    0.00    0.00   0.00

　　sdb          0.00   1122  17.00  9.00  192.00 9216.00    96.00  4608.00   123.79   137.23 1033.43  13.17 100.10

　　sdc          0.00    0.00     0.00   0.00     0.00     0.00      0.00     0.00     0.00             0.00    0.00      0.00   0.00
```
如果 iostat没有，需要yum install sysstat　　   


如上数据查看可得：  
```
%util 100.10     %idle 66.29   
```

- %util 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈，
- %idle小于70% IO压力就较大了,读取速度会出现较多的wait。

- iostat -dx 显示磁盘扩展信息r/s 和 w/s 分别是每秒的读操作和写操作，而rKB/s 和wKB/s 列以每秒千字节为单位显示了读和写的数据量。   
- 两对数据值都很高的话说明磁盘io操作是很频繁。


#### vmstat
- vmstat 命令报告关于线程、虚拟内存、磁盘、陷阱和 CPU 活动的统计信息。   
由 vmstat 命令生成的报告可以用于平衡系统负载活动。系统范围内的这些统计信息(所有的处理器中)都计算出以百分比表示的平均值，或者计算其总和。

- 输入命令：vmstat 2 5  
- 如果发现等待的进程和处在非中断睡眠状态的进程数非常多，并且发送到块设备的块数和从块设备接收到的块数非常大，那就说明磁盘io比较多。

- vmstat参数解释： 
```
Procs
　　r: 等待运行的进程数 
    b: 处在非中断睡眠状态的进程数 
    w: 被交换出去的可运行的进程数。
    此数由 linux 计算得出，但 linux 并不耗尽交换空间

Memory
　　swpd: 虚拟内存使用情况，单位：KB
　　free: 空闲的内存，单位KB
　　buff: 被用来做为缓存的内存数，单位：KB
Swap
　　si: 从磁盘交换到内存的交换页数量，单位：KB/秒
　　so: 从内存交换到磁盘的交换页数量，单位：KB/秒
IO
　　bi: 发送到块设备的块数，单位：块/秒
　　bo: 从块设备接收到的块数，单位：块/秒
System
　　in: 每秒的中断数，包括时钟中断
　　cs: 每秒的环境(上下文)切换次数
CPU
　　按 CPU 的总使用百分比来显示
　　us: CPU 使用时间
　　sy: CPU 系统使用时间
　　id: 闲置时间
```