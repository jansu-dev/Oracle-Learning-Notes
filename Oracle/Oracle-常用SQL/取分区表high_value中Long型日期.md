#### 文章概览
- [是什么？](#是什么？)  
- [为什么？](#为什么？)  
- [怎么做？](#怎么做？)  
- [参考文章](#参考文章)



### 是什么？  
&nbsp;&nbsp;&nbsp;&nbsp;在日常运维工作中，免不了要编写一些运维脚本来了帮助我们完成自动化运维工作；  
&nbsp;&nbsp;&nbsp;&nbsp;本片文章提供的内容，在检测表空间容量不足时，可自动删除规定日期的表分区；   
&nbsp;&nbsp;&nbsp;&nbsp;将分区表信息查询视图user_tab_partitions的LONG型字段转换为varchar2型，可查看的数据，进而取出分区表创建日期。



### 为什么？  

1. ##### 因为数据库删除规定日期内的分区，一定需要查询分区表的创建日期；
2. ##### high_value的字段类型又是Long型的，无法直接查看   

- 网上有一种解决办法是使用to_lob函数现将high_value字段数据存到一个中间表； 
- 再从中间表使用substr函数转换中间表中的high_value字段数据到另一个中间表；
- 最后查询中间表中两次转换的数据，拼接删除语句，触发job执行删除分区。

3. ##### 2中给出的方案需要两个中间表，不便于移植。



### 怎么做？  

1. 鉴于以上“为什么？”给出的原因，我创建了一个函数func_longtovarchar2，出入表名和分区名直接将该分区表所有创建一起直接收取出来，以varchar2类型返回。  
2. 编写函数执行DML语句输入到变量中，使用substr函数对变量进行处理后将数据返回。  

##### 转换函数（long型-->varchar2型）
```
CREATE OR REPLACE function func_longtovarchar2 (v_tablename varchar2,v_partitionname varchar2) return varchar2 is
  v_sql varchar2(2000);
  v_chardate varchar2(1000);
begin
  v_sql := 'select high_value from user_tab_partitions where table_name= '||''''||upper(v_tablename)||''''||' and partition_name='||''''||upper(v_partitionname)||'''';
  execute immediate v_sql into v_chardate;
  v_chardate := substr(v_chardate, 1, 500);
  RETURN v_chardate;
end;
/
```
3. 最后附上检测删除分区的脚本，大家可根据自己的需求在此基础上自行修改。 

##### 删分区存储过程
```
v_tablename 分布表名  
v_firstpar  初始化分区名  
v_keep_day  保存分区日期数（单位/天）


CREATE OR REPLACE PROCEDURE delete_partition (v_tablename in varchar2,v_firstpar in varchar2,v_keep_day in number) IS
  TYPE kv_tbname_date IS RECORD(parname varchar2(40),vdate varchar2(40));
  kv_par kv_tbname_date;
  v_exec_sql varchar2(400);
  v_tmp_date date;
  v_upper_firstpar varchar2(40);
  v_delete_date date;
  CURSOR mycursor(tb_name varchar2) IS select TT.partition_name,substr(func_longtovarchar2(TT.table_name,TT.partition_name),10,20) from user_tab_partitions TT where table_name = upper(tb_name);
begin
  select trunc(sysdate)-v_keep_day into v_delete_date from dual;
  select upper(v_firstpar) into v_upper_firstpar from dual;
  open mycursor('TESTTABLE');
    FETCH mycursor into kv_par;
    while mycursor%FOUND LOOP
      --筛选中间变量分区表创建日期
      select TO_DATE(kv_par.vdate, 'YYYY-MM-DD HH24:MI:SS') into v_tmp_date from dual;
      if v_tmp_date < v_delete_date then
        if kv_par.parname = v_upper_firstpar and v_firstpar is not null then
          --如果init分区不能删直接跳过
          --dbms_output.put_line('skip...');
          FETCH mycursor into kv_par;
        end if;
        v_exec_sql := 'ALTER TABLE ' || v_tablename || ' DROP PARTITION ' || kv_par.parname;
        --dbms_output.put_line(v_exec_sql);
        execute immediate v_exec_sql;
      end if;
      FETCH mycursor into kv_par;
    END LOOP;
  close mycursor;
end;
/
```






