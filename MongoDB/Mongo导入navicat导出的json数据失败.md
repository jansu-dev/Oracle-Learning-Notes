
​		工作中面对用户复杂的环境，往往会出现，从**向日葵**远程到**跳板机**再到**数据库服务器**的尴尬现象，甚至还会嵌套多层，这就给数据迁移带来了不便。下面价绍最近一次，mongo的数据迁移遇到开发人员贪图<span style='color:red'>省事</span>使用navicat工具导出json格式数据，使用mongoimport工具导入时引发问题的全过程。主要内容如下：

* [还原导出过程](还原导出过程)
* [定位问题](定位问题)
* [问题产生原因](问题产生原因)
* [解决问题](解决问题)

![mongo01](http://cdn.lifemini.cn/dbblog/20210115/4a515936d7364bae806459d0bd838f30.png)


------





### 1.还原导出过程

下面简要介绍了使用navicat工具导出mongo数据的全过程，

首先，进入导出向导；

![mongo01](E:\Oracle\整理\MongoDB\Mongo导入navicat导出的json数据失败\01.png)

其次，选择导出文件格式；

![mongo02](http://cdn.lifemini.cn/dbblog/20210115/8577aecd8f9240c09efe92e4cfae09cb.png)

选择导出文件格式；

![mongo03](http://cdn.lifemini.cn/dbblog/20210115/a3bc3376fd3b48f188563331bc322590.png)

选择文件导出字段；

![mongo04](http://cdn.lifemini.cn/dbblog/20210115/75d92dc15d2f415291bde98a92a30d8f.png)
![mongo05](http://cdn.lifemini.cn/dbblog/20210115/6b93ad84a2ba4c84aadc1324d672162d.png)
![mongo06](http://cdn.lifemini.cn/dbblog/20210115/edac0bc3ba6c4a5d8548a02963682cd6.png)

开始导出；

![mongo07](http://cdn.lifemini.cn/dbblog/20210115/255d9f998fa9466c8d1fb9f03cfac9e8.png)

看到successfully日志，说明导出成功。

![mongo08](http://cdn.lifemini.cn/dbblog/20210115/0e5f99c0358b479981e5169e87e2dd25.png)





### 2.定位问题

```
mongoimport -u XXXX -p XXXX -d coresu -c restaurants --type=json 
--file=/home/mongodump/exp_restaurants.json --drop
```
生产库执行上述命令后，日志显示文档已经成功导入，但是在应用端检索不到数据，十分不解。
抱着试一试的心态，执行了db.restaurant.find().count()显示为结果为1，
<span style='color:red'>这很奇怪，导入数据按理来说不应该只有一条数据啊！！！</span>
在经过与数据库开发人员的求证后，该集合确实不只是一行数据，判定问题在数据集。





### 3.问题产生原因

打开导出的json文件可以看到，如果使用navicat工具导出数据会自动给数据最外层加上
```
{
  "RECOREDS" : [
    ...
    ...
    ...
  ]
}
```
而且，每条json格式的记录之间还会自动添加逗号。

![mongo09](http://cdn.lifemini.cn/dbblog/20210115/55f98023976f439cb75aa94de5e7755e.png)

正常使用mongoexport命令导出的json文件格式如下：

![mongo10](http://cdn.lifemini.cn/dbblog/20210115/63032a1ea9d44186ab3489c8428f167a.png)

<span style='color:red'>注意：</span>

<span style='color:red'>1.mongoimport导入json数据格式文件，要求不能有最外层不能有多余包含；</span>

<span style='color:red'>2.每条json数据之间，不以逗号分隔。
</span>





### 4.解决问题

```
mongoexport -u XXXX -p XXXX -d coresu -c restaurants --type=json 
-o /home/mongodump/exp_restaurants.json
```
最后使用mongo官方工具mongoexport导出，传到生产库成功导入。

