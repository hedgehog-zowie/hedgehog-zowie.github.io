---
title: spark-trouble-registerTempTableNotFound
date: 2016-09-01 00:00:00
toc: true
tags:
- spark trouble
categories:
- spark
---

# 问题

spark dataframe提供的registerTempTable方法可以将数据抽象成临时表，方便用户对数据进行操作。但今天使用该方法时注册临时表后进行查询遇到错误：`table not found`，如下：

```
18/05/22 15:27:51 INFO parse.ParseDriver: Parsing command: select * from t
18/05/22 15:27:51 INFO parse.ParseDriver: Parse Completed
org.apache.spark.sql.AnalysisException: Table not found: t; line 1 pos 14
```

# 原因

由于疏忽，查询时使用的sqlContext不是注册临时表时使用的sqlContext，如下：

```scala
var hc1 = new HiveContext(sc)
var t = hc1.sql("select * from test_mac limit 10");
t.registerTempTable("t"); // HiveContext是SqlContext的子类，该方法会调用SqlContext类中的createDataFrame方法创建临时表
var hc2 = new HiveContext(sc)
hc2.sql("select * from t").show() // 导致错误
```

# 延伸

DataFrame类中的registerTempTable方法代码如下：

```scala
  def registerTempTable(tableName: String): Unit = {
    sqlContext.registerDataFrameAsTable(this, tableName)
  }
```

从这段代码可以看出，dataFrame注册的临时表是使用sqlContext对象调用registerDataFrameAsTable方法来实现的，也就是说临时表是与sqlContext对象强相关的，因此在一个sqlContext对象中注册的临时表，使用另一个sqlContext对象来操作，是访问不到的，即出现`table not found`错误。

