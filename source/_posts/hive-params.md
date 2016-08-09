---
title: hive-params
date: 2016-07-15 10:08:05
toc: true
tags: 
- hive
categories: 
- hive
---

本文主要翻译自hive 1.2.1版本的hive-default.xml.template。

# hive参数配置

Hive提供三种可以改变环境变量的方法，分别是：（1）、修改配置文件；（2）、命令行参数；（3）、进入cli后设置参数。

## 修改配置文件

修改Hive的配置文件：`${HIVE_HOME}/conf/hive-site.xml`

## 命令行参数

在命令行添加-hiveconf param=value来设定参数
如：
```
hive --hiveconf mapreduce.job.queuename=queue1
```

## 进入cli后设置参数

可以在进入cli后，使用SET设置参数，如：

```
hive> set mapreduce.job.queuename=queue1;
```

这种配置也是对本次启动的会话有效，下次启动需要重新配置。
在HQL中使用SET关键字还可以查看配置的值，如下：

```
hive> set mapreduce.job.queuename;
mapreduce.job.queuename=queue1
```

如果set后面什么都不添加，这样可以查到Hive的所有属性配置。

# hive参数大全

参数|说明|默认值
--|--|--
11|12|13
21|22|23
31|32|33