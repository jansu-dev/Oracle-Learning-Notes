---
title:[Oracle]--触发器加密函数
date:2020-09-11
---



##### 利用数据库触发器和函数给数据库内容加密。



### 加密函数一

例：给数据库183添加加密函数
YOURPASSWORD:手动设置的密码；

```
create function encrypt_des_183(p_text varchar2) return varchar2 is
  v_text varchar2(4000);
begin
  if p_text is null then
    return null;
  end if;
  v_text := UTL_I18N.STRING_TO_RAW(p_text, 'AL32UTF8');
  v_text := ENCRYPT_DES(v_text, 'YOURPASSWORD');
  return v_text;
end;
```



### 加密函数二

例：给数据库183添加解密函数

```
create  function decrypt_des_183(p_text varchar2,p_key varchar2) return varchar2 is
  v_text varchar2(2000);
begin
  if p_text is null then
    return null;
  end if;
  v_text := DECRYPT_DES(p_text, p_key);
  v_text := UTL_I18N.RAW_TO_CHAR(v_text, 'AL32UTF8');
  return v_text;
exception
  when others then
    RETURN p_text;
end;
```