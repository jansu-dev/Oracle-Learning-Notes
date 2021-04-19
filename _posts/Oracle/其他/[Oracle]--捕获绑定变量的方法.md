---
title:[Oracle]--捕获绑定变量的方法
date:2020-06-19
---



##### 捕获绑定变量

```
select sql_id,
        name,
        datatype_string,
        case datatype
          when 180 then --TIMESTAMP
           to_char(ANYDATA.accesstimestamp(t.value_anydata),
                   'YYYY/MM/DD HH24:MI:SS')
          else
           t.value_string
        end as bind_value,
        last_captured
   from v$sql_bind_capture t
  where sql_id = '6mvu1s81yvhpx';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('SQL_ID',NULL,' ADVANCED ALLSTATS LAST PEEKED_BINDS'));
可以获取绑定变量,前提是还没被清出sga.
```