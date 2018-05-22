---
title: spark-tips-autobroadcastjoin
date: 2016-09-02 00:00:00
toc: true
tags:
- spark tips
categories:
- spark
---

spark 中

使用spark sql时，spark会自动进行一些优化，autobroadcastjoin就是其中一项，该功能由参数`spark.sql.autoBroadcastJoinThreshold `控制，默认为10000000（10m）。

todo: 其他优化项。