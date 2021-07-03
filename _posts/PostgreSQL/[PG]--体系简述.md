---
title:[PG]--体系简述
date:2020-10-14
---



##### [PG]--体系简述

# PostgreSQL

PG体系结构
PG存储架构
PG网络与字符集
PG备份恢复
SQL调优
流复制 与 PG Pool



### PG体系结构

#### 关键词标识符

1. 一个命令由一个记号的序列构成，并由一个分号结尾;
2. 系统中一个标识符的长度不能超过 NAMEDATALEN-1 字节，在命令中可以写超过此长度的标识符，但是它们会被截断。默认情况下，NAMEDATALEN 的值为64，因此标识符的长度上限为63字节。如果这个限制有问题，可以在src/include/pg_config_manual.h中修改NAMEDATALEN 常量;
3. 关键词和不被引号修饰的标识符是大小写不敏感的。
4. 或被引号修饰的标识符符总是一个标识符而不会是一个关键字,例：select * from "select"可以，但是select * from select却不行;
5. 如果两个双引号内部要包含一个双引号，则写两个双引号转义。

#### 常量与变量

三种隐式类型常量：字符串、位串和数字。

```
SELECT 'foo'
'bar';
等同于：
SELECT 'foobar';
但是：
SELECT 'foo' 'bar';
则不是合法的语法
```

“美元引用”方式书写字符串常量

```
$$Dianne's horse$$
$SomeTag$Dianne's horse$SomeTag$
```

在一个美元引用字符串中不需要对字符进行转义：字符串内容总是按其字面意思写出;
反斜线不是特殊的，并且美元符号也不是特殊的;
常用方法

```
$function$
BEGIN
 RETURN ($1 ~ $q$[\t\r\n\v\\]$q$);
END;
$function$
```

标签是大小写敏感的，因此$tag$String content$tag$是正确的，但是$TAG$String content$tag$不正确

位串常量
例如B'1001'。位串常量中允许的字符只有0和1
位串常量可以用十六进制记号法指定，使用一个前导X（大写或小写形式）,例如X'1FF'。
强制类型转换

```
REAL '1.23' -- string style
1.23::REAL -- PostgreSQL (historical) style


type 'string' 'string'::type
CAST ( 'string' AS type )
```



### 数组构造器

一个数组构造器是一个能构建一个数组值并且将值用于它的成员元素的表达式。一个简单的
数组构造器由关键词ARRAY、一个左方括号[、一个用于数组元素值的表达式列表（用逗号
分隔）以及最后的一个右方括号]组成。例如：

```
SELECT ARRAY[1,2,3+4];
 array
---------
 {1,2,7}
(1 row)
```

你可以构造一个空数组，但是因为无法得到一个无类型的数组，你必须显式地把你的空数组
造型成想要的类型



### 行构造器

默认情况下，由一个ROW表达式创建的值是一种匿名记录类型。如果必要，它可以被造型
为一种命名的组合类型 — 或者是一个表的行类型，或者是一种用CREATE TYPE AS创建的
组合类型。为了避免歧义，可能需要一个显式造型。例如：

```
CREATE TABLE mytable(f1 int, f2 float, f3 text);
CREATE FUNCTION getf1(mytable) RETURNS int AS 'SELECT $1.f1' LANGUAGE SQL;
-- 不需要造型因为只有一个 getf1() 存在
SELECT getf1(ROW(1,2.5,'this is a test'));
 getf1
-------
 1
(1 row)
CREATE TYPE myrowtype AS (f1 int, f2 text, f3 numeric);
CREATE FUNCTION getf1(myrowtype) RETURNS int AS 'SELECT $1.f1' LANGUAGE SQL;
-- 现在我们需要一个造型来指示要调用哪个函数：
SELECT getf1(ROW(1,2.5,'this is a test'));
ERROR: function getf1(record) is not unique
SELECT getf1(ROW(1,2.5,'this is a test')::mytable);
 getf1
-------
 1
(1 row)
SELECT getf1(CAST(ROW(11,'this is a test',2.5) AS myrowtype));
 getf1
-------
 11
(1 row)
```

行构造器可以被用来构建存储在一个组合类型表列中的组合值，或者被传递给一个接受组合
参数的函数。还有，可以比较两个行值，或者用IS NULL或IS NOT NULL测试一个行，例
如：

```
SELECT ROW(1,2.5,'this is a test') = ROW(1, 3, 'not the same');
SELECT ROW(table.*) IS NULL FROM table; -- detect all-null rows
```



### 表达式计算规则

子表达式的计算顺序没有被定义。特别地，一个操作符或函数的输入不必按照从左至右或其
他任何固定顺序进行计算。
此外，如果一个表达式的结果可以通过只计算其一部分来决定，那么其他子表达式可能完全
不需要被计算。例如，如果我们写：
SELECT true OR somefunc();
那么somefunc()将（可能）完全不被调用。如果我们写成下面这样也是一样：
SELECT somefunc() OR true;
注意这和一些编程语言中布尔操作符从左至右的“短路”不同。
因此，在复杂表达式中使用带有副作用的函数是不明智的。在WHERE和HAVING子句中依
赖副作用或计算顺序尤其危险，因为在建立一个执行计划时这些子句会被广泛地重新处理。
这些子句中布尔表达式（AND/OR/NOT的组合）可能会以布尔代数定律所允许的任何方式被
重组。
当有必要强制计算顺序时，可以使用一个CASE结构（见第 9.17 节）。例如，在一
个WHERE子句中使用下面的方法尝试避免除零是不可靠的：
SELECT ... WHERE x > 0 AND y/x > 1.5;
但是这是安全的：
SELECT ... WHERE CASE WHEN x > 0 THEN y/x > 1.5 ELSE false END;
一个以这种风格使用的CASE结构将使得优化尝试失败，因此只有必要时才这样做（在这个
特别的例子中，最好通过写y > 1.5*x来回避这个问题）。
不过，CASE不是这类问题的万灵药。上述技术的一个限制是， 它无法阻止常量子表达式的
提早计算。如第 37.6 节 中所述，当查询被规划而不是被执行时，被标记成 IMMUTABLE的
函数和操作符可以被计算。因此
SELECT CASE WHEN x > 0 THEN x ELSE 1/0 END FROM tab;
很可能会导致一次除零失败，因为规划器尝试简化常量子表达式。即便是 表中的每一行都
有x > 0（这样运行时永远不会进入到 ELSE分支）也是这样。
虽然这个特别的例子可能看起来愚蠢，没有明显涉及常量的情况可能会发生 在函数内执行的
查询中，因为因为函数参数的值和本地变量可以作为常量 被插入到查询中用于规划目的。例
如，在PL/pgSQL函数 中，使用一个IF-THEN-ELSE语句来 保护一种有风险的计算比把它嵌在
一个CASE表达式中要安全得多。
另一个同类型的限制是，一个CASE无法阻止其所包含的聚合表达式 的计算，因为在考
虑SELECT列表或HAVING子句中的 其他表达式之前，会先计算聚合表达式。例如，下面的
查询会导致一个除零错误， 虽然看起来好像已经这种情况加以了保护：
SELECT CASE WHEN min(employees) > 0
THEN avg(expenses / employees)
END
FROM departments;
min()和avg()聚合会在所有输入行上并行地计算， 因此如果任何行有employees等于零，
在有机会测试 min()的结果之前，就会发生除零错误。取而代之的是，可以使用 一 个WHERE或FILTER子句来首先阻止有问题的输入行到达 一个聚合函数。