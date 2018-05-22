---
title: hadoop-tips-when-reduce-start
date: 2016-09-23 08:50:52
toc: true
tags:
- hadoop
categories:
- hadoop
---

# reduce的启动时机

[Yarn源码分析之参数mapreduce.job.reduce.slowstart.completedmaps介绍](http://blog.csdn.net/lipeng_bigdata/article/details/51285687)

[mapreduce作业reduce被大量kill掉](http://blog.csdn.net/bigdatahappy/article/details/41950909)

MRAppMaster在确定reduce task启动时机时，需要自己设计资源申请策略以防止因reduce task过早启动照成资源利用率低下和map task因分配不到资源而饿死。
MRAppMaster在MRv1原有策略（map task完成数目达到一定比例后才允许启动reduce task）基础上添加了更为严格的资源控制策略和抢占策略，这里主要涉及到以下三个参数：
mapreduce.job.reduce.slowstart.completedmaps：当map task完成的比例达到该值后才会为reduce task申请资源，默认是0.05。

yarn.app.mapreduce.am.job.reduce.rampup.limit：在map task完成之前，最多启动reduce task比例，默认是0.5

yarn.app.mapreduce.am.job.reduce.preemption.limit：当map task需要资源但暂时无法获取资源（比如reduce task运行过程中，部分map task因结果丢失需重算）时，
为了保证至少一个map task可以得到资源，最多可以抢占reduce task比例，默认是0.5

如果上面三个参数设置的不合理可能会出现提交的job出现大量的reduce被kill掉，
因此MRAppMaster自己需要设计资源申请策略以防止因reduce task过早启动照成资源利用率低下和map task因分配不到资源而饿死，然后通过抢占机制，大量reduce任务被kill掉。
可以合理调节上面三个配置参数来消除这种情况。