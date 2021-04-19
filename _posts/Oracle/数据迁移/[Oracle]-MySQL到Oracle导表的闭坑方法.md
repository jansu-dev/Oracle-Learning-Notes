---
title: [Oracle]-MySQL到Oracle导表的闭坑方法
date: 2020-11-12
---



### 修改mysqldump导出文件

```

```



### 不兼容字段修改

```
:%s/\n/\rl\r/g
875  ll -lrth
  876  pwd
  877  ifconfig
  878  ip addr
  879  ifconfig
  880  exit
  881  ll
  882  ll -lrth
  883  mkdir szp_20201112
  884  ll
  885  mv jigouxinxi1112.zip szp_20201112/
  886  ll
  887  cd szp_
  888  cd szp_20201112/
  889  ll
  890  unzip jigouxinxi1112.zip
  891  ll
  892  vi bic_base_org1112.sql
  893  vi bic_base_org1112.sql
  894  ll -lrth
  895  vi bic_org_assist1112.sql
  896  vi bic_org_assist1112.sql
  897  vi bic_org_assist1112.sql
  898  ll -lrth
  899  vi bic_base_org1112.sql
  900  vi bic_org_assist1112.sql
  901  lsnrctl status
  902  ifconfig
  903  exit
  904  sqlplus cp_os2019/os
  905  sqlplus cp_os2019/123456
  906  exit
  907  lsnrctl status
  908  sqlplus cp_os2019/123456
  909  cd szp_20201112/
  910  ll
  911  sqlplus cp_os2019/123456
  912  ll -lrth
  913  vi szp_bic_base_org1112_spool.lst
  914  rm szp_bic_base_org1112_spool.lst
  915  sqlplus cp_os2019/123456
  916  ll
  917  vi base.lst
  918  more base.lst
  919  ll
  920  cat base.lst
  921  cat base.lst |grep -v ^$|grep -v ^已
  922  ll
  923  rm base.lst
  924  sqlplus cp_os2019/123456
  925  cat base.lst |grep -v ^$|grep -v ^已
  926  cat bic_base_org1112.sql |grep -v ^$|grep -v ^c
  927  cat bic_base_org1112.sql |grep -v ^$|grep -v ^c|wc -l
  928  cat bic_base_org1112.sql |grep -v ^$|grep -v ^commit;|wc -l
  929  cat bic_base_org1112.sql |grep -v ^$|grep -v ^commit|wc -l
  930  cat bic_base_org1112.sql |grep -v ^$|grep -v ^commit|grep -v ^i
  931  vi bic_base_org1112.sql
  932  cat bic_base_org1112.sql |grep -v ^$|grep -v ^commit|grep -v ^I
  933  cat bic_base_org1112.sql |grep -v ^$ |grep -v ^commit|grep -v ^I
  934  cat bic_base_org1112.sql |grep -v ^commit|grep -v ^I
  935  cat bic_base_org1112.sql |grep -v ^commit|grep -v ^I
  936  cat bic_base_org1112.sql |grep -v ^$ |grep -v ^commit|grep -v ^I |wc -l
  937  cat bic_base_org1112.sql |grep -v ^$ |grep -v ^commit|grep -v ^I
  938  cat bic_base_org1112.sql |grep -v ^$ |grep -v ^commit|wc -l
  939  sqlplus cp_os2019/123456
  940  ll
  941  vi org.lst
  942  more org.lst
  943  more org.lst
  944  more org.lst
  945  less org.lst
  946  ll
  947  ll
  948  rm org.lst
  949  sqlplus cp_os2019/123456
  950  rm org.lst
  951  sqlplus cp_os2019/123456
  952  ll -lrth
  953  cat base.lst |grep -v ^$|grep -v ^已|less
  954  cat org.lst |grep -v ^$|grep -v ^已|less
  955  cat org.lst |grep -v ^$|grep -v ^已|lwc -l
  956  cat org.lst |grep -v ^$|grep -v ^已|wc -l
  957  cat org.lst |grep -v ^$|grep -v ^已|less
  958  ll -lrth
  959  rm org.lst
  960  sqlplus cp_os2019/123456
  961  rm org.lst
  962  sqlplus cp_os2019/123456
  963  ll -lrth
  964  less org.lst
  965  cat org.lst |grep -v ^$|grep -v ^已|wc -l
  966  cat org.lst |grep -v ^$|grep -v ^已
INSERT INTO JAN_TESTTB2_20201111 VALUES (...) ;

INSERT INTO JAN_TESTTB2_20201111 VALUES (...) ;

INSERT INTO JAN_TESTTB2_20201111 VALUES (...) ;

INSERT INTO JAN_TESTTB2_20201111 VALUES (...) ;

INSERT INTO JAN_TESTTB2_20201111 VALUES (...) ;
```



### DBlink导表

```

```



### 规范更改索引

```

```

- 收集统计信息

```
begin
dbms_stats.gather_table_stats(ownname => 'JAN_USER',
                              tabname => 'JAN_TESTTB1_20201112',
                              cascade => TRUE,
                              estimate_percent => dbms_stats.auto_sample_size,
                              method_opt => 'for all columns size auto',
                              degree =>8);
end;


JAN_TESTTB2_20201112
JAN_TESTTB1_20201112


alter table JAN_TESTTB2 rename to JAN_TESTTB2_bak;

alter table JAN_TESTTB2_20201112 rename to JAN_TESTTB2;

alter table JAN_TESTTB2_bak rename to JAN_TESTTB2_20201112;


alter table JAN_TESTTB1 rename to JAN_TESTTB1_bak;

alter table JAN_TESTTB1_20201112 rename to JAN_TESTTB1;

alter table JAN_TESTTB1_bak rename to JAN_TESTTB1_20201112;


select count(*) from JAN_TESTTB2;
select count(*) from  JAN_TESTTB1;
```

