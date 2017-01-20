---
title: sqoop-trouble-shooting
date: 2016-10-14 17:37:51
toc: true
tags:
- spring spring-data hadoop
categories: 
- spring
---

记录使用Spring过程中遇到的问题、原因及解决办法。

# NoClassDefFoundError: org/apache/hadoop/conf/Configuration$DeprecationDelta

使用spring-data-hadoop-hbase时遇到如下错误：
`Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/conf/Configuration$DeprecationDelta`

原因：spring-data-hadoop-hbase版本为2.2.0，其中使用的hadoop-hdfs版本为2.6.0，因此出现如上错误。

解决办法：使用同版本的hadoop包。

PS:若遇到错误：org.eclipse.jdt.internal.compiler.CompilationResult.getProblems，这是由于jsp解析包依赖冲突所致，将其从hadoop/hive/hbase等相关的包中exclude掉即可。