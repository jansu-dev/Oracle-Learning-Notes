#### 文章概览
- [Oracle数据库版本](#Oracle数据库版本)  
- [是什么？](#是什么？) 
- [为什么？](#为什么？)  
- [怎么做？](#怎么做？)  
- [参考文章](#参考文章)



### Oracle数据库版本  

&nbsp;&nbsp;&nbsp;&nbsp;操作系统：   
&nbsp;&nbsp;&nbsp;&nbsp;OGG版本号：   
&nbsp;&nbsp;&nbsp;&nbsp;Oracle数据库版本：



### 是什么？  

&nbsp;&nbsp;&nbsp;&nbsp;运维工作中，经核查那刚出现数据库运行缓慢的问题，其中百分之九十是SQL出现了问题。  
&nbsp;&nbsp;&nbsp;&nbsp;那么就需要我们对SQL进行调优了，为了快速恢复业务强制指定执行计划使我们最常使用的一种临时解决SQL问题的手段。



### 为什么？

因为hint可以强制执行计划，所以可作为一种改变执行计划参考方式。
##### 以下为多表连接时使用hint方法强制执行计划几种情况
##### 强制使用hash连接
```
select /*+ use_hash(e d)*/ ename,loc from emp e ,dept d where e.deptno=d.deptno;
```

- 注意： 
    - e和d在括号中的顺序不会影响到SQL的执行顺序

##### 强制使用merge连接
```
select /*+ use_merge(e d)*/ ename,loc from emp e ,dept d where e.deptno=d.deptno;
```

##### 强制使用nested loops连接
```
select /*+ use_nl(e d)*/ ename,loc from emp e ,dept d where e.deptno=d.deptno;
```

##### 强制使用access full扫描
```
select /*+ full(emp)*/ * from emp;
```

##### 强制使用主键扫描
```
select /*+ index(e pk_emp)*/ * from emp e;
```

- 注意： 
    - 注意查询表如果使用了别名，在hint中一定要使用别名否则失效。

##### 强制并行提高full scan效率
```
select /*+ full(emp) parallel(emp 4)*/ * from emp; 
select /*+ parallel(emp 4)*/ * from emp; 
```
- 注意：
    - 指定并行方式默认会以full scan的方式进行扫描。

```
SCOTT@ora10g> select /*+ parallel(emp 2)*/* from emp; 
```



### 参考文章

- [参考文章1-Oracle11g中强制执行计划4中方式](http://blog.itpub.net/29785807/viewspace-2643074/)  
- [参考文章2-SQL profile使用方法](https://blog.csdn.net/AikesLs/article/details/86246091)  
- [参考文章3-SPM使用方法](https://www.cnblogs.com/xibuhaohao/p/10271503.html)