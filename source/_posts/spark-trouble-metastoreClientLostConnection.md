---
title: spark-trouble-metastoreClientLostConnection
date: 2016-09-03 00:00:00
toc: true
tags:
- spark trouble
categories:
- spark
---

# 问题

使用spark-sql操作hive的数据时，遇到错误如下：

```
RetryingMetaStoreClient: MetaStoreClient lost connection. Attempting to reconnect.
org.apache.thrift.TApplicationException: Invalid method name: 'alter_table_with_cascade'
	at org.apache.thrift.TApplicationException.read(TApplicationException.java:111)
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:71)
```

# 原因

spark编译用的hive版本与服务器上安装的hive版本不一致。

# 解决办法

```
在classpath里加入hive0.13.jar
在spark-defaults.conf中加入以下配置
spark.sql.hive.metastore.version 0.13.1
spark.sql.hive.metastore.jars /opt/cloudera/parcels/CDH/lib/hive/lib/*:/opt/cloudera/parcels/CDH/lib/hadoop/client/*
```



