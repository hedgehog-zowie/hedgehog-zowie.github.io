---
title: mysql-tips
date: 2016-08-09 18:31:24
toc: true
tags:
- mysql
categories:
- mysql
---
记录使用mysql的过程中一些技巧、方法、注意事项。

# on case when
做关联查询时需要根据输入数据的值来判断关联条件。
如有表t1(code, name)、t2(code,name)，当code为空时，使用name关联。
在查询的时候，这种情况一般都使用case when来进行逻辑判断，因此突发奇想是否可以在表关联时使用case when, 即
```
select * from t1 left join t2 on 
case when t1.code is null then t1.name = t2.name 
else t1.code = t2.code end
```
这种用法是错误的，尽管其在mysql下并不会报错，case when放在on后面，不会进行操作。
正确的写法应该是：
```
select * from t1 left join t2 on
(t1.code is null and t1.name = t2.name)
or
(t1.code = t2.code)
```
