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

# collate
使用关联查询已经过滤掉了某表中不存在的记录，而在往该表中插入这些数据时，报出主键冲突，如下：
select a.* from a left join b on a.id = b.id and b.id is null;
经查，由于mysql中默认不区分大小写，而表a中id字段的校对集设置为`utf8_bin`(将字符串中的每一个字符用二进制数据存储，区分大小写)，而表b中的id校对集为默认的`utf8_general_ci`(ci为case insensitive的缩写，即大小写不敏感，ps:还有一个校对集`utf8_general_cs`——cs为case sensitive的缩写，即大小写敏感)，因此在关联的时候区分了大小写，而导致不相等，而在插入时又不区分大小写，导致主键冲突。
`注意：子查询中的校对集与所查询的表中一致`

