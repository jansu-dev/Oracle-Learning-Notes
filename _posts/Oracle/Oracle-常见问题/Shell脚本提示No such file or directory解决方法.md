
#### 解决方法

1. ##### Windows下编码格式转换：

    + 用编辑器工具先将脚本编码转换为unix，再放到Linux中执行。

2. ##### Linux下编码格式转换：

    + 用vi打开该sh文件,会看到底部出现fileformat=dos字样
    + 输入:set ff=unix  重新设置下文件格式后保存退出，再执行就OK了。

3. ##### linux下确保文件有可执行权限：

    + chmod u+x filename 