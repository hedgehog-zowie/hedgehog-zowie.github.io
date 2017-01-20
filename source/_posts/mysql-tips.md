---
title: mysql-tips
date: 2016-09-23 08:50:52
toc: true
tags:
- mysql
categories:
- mysql
---

记录使用mysql的过程中一些技巧、方法、注意事项。

# 浅谈MySQL之 Handler_read_*参数 

原文：[浅谈MySQL之 Handler_read_*参数 ](http://gfsunny.blog.51cto.com/990565/1558480)

摘要：

使用步骤如下：
```
1.FLUSH STATUS;
2.SELECT ...;
3.SHOW SESSION STATUS LIKE 'Handler_read%';
4.EXPLAIN SELECT ...;
```

```
Handler_read_first: first key被读取的次数，如果该值很高，则说明做了很多全索引扫描，例如：SELECT col1 FROM foo, 假设列col1是索引列。

Handler_read_key: 基于索引请求数据的次数，如果该值很高，则说明查询高效地使用了索引。

Handler_read_last：last key被读取的次数，如果存在ORDER BY子句，服务器将会在几个next-key请求后发出一个first-key请求，然而在ORDER BY DESC子句下，服务器将会在几个previous-key后发出一个last-key请求，该变量在MySQL 5.6.1版本加入。

Handler_read_next：按照索引从数据文件中取出数据的次数。

Handler_read_prev：按照索引倒序从数据文件中取出数据的次数，一般就是ORDER BY ... DESC。注意Handler_read_next是ORDER BY ... ASC的方式取数据。

Handler_read_rnd：基于混合位置请求数据的次数，如果该值很高，则说明做了很多要求结果排序的查询，可能查询要求全表扫描或join操作未正确使用key。

Handler_read_rnd_next：在数据文件中读下一行的请求数。如果你正进行大量的表扫描，该值较高。通常说明你的表索引不正确或写入的查询没有利用索引。

```


# mysql索引失效的几种情况

原文：[mysql索引失效的几种情况](http://www.jb51.net/article/50649.htm)


