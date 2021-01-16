##### 存储过程工作：批量删除数据的存储过程，每100行提交一次。  

##### 执行代码： 
```
create or replace procedure del_item_tab is
  objid rowid;
  cursor item_del is
    select rowid
      from lightworkflow_item it
     where exists (select 1
              from light_workflow_dispatchticket t
             where t.ticket_id = it.entity_id
               and t.current_stage = 'ticket_close');

begin
  open item_del;
  loop
    fetch item_del into objid;
        exit when item_del%notfound;
        delete from lightworkflow_item where rowid = objid;
        if mod(item_del%rowcount, 100) = 0 then
            commit;
        end if;
    end loop;
  commit;
  close item_del;
end;

```