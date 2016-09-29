---
title: hive-trouble-shooting
date: 2016-09-27 14:44:02
toc: true
tags: 
- hive
categories: 
- hive
---

此处记录使用Hive过程中遇到的问题、原因及解决办法。

# HiveServer提交任务时报出GC overhead limit exceeded

## 问题描述

由kettle连接HiveServer，查看hive.log可以看到已经接收到任务，但随即立即报出错误如下：
```
2016-08-25 09:04:10,671 ERROR [pool-1-thread-106]: ql.Driver (SessionState.java:printError(545)) - FAILED: Execution Error, return code -101 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask. GC overhead limit exceeded
```
且没有打印出堆栈信息。

##　原因
由GC overhead limit exceeded可知，是由于花费了大量的时间进行垃圾回收所致，原文如下：
```
if too much time is being spent in garbage collection: if more than 98% of the total time is spent in garbage collection and less than 2% of the heap is recovered, an OutOfMemoryError will be thrown
```
简单来说，就是内存不够，导致了大量GC，而JVM中实际上需要使用很多内存，从来垃圾回收之后并不能释放很多空间，如此反复。

查看hive中的源码，ql.Driver类中的execute方法启动任务，如下：
```
TaskRunner runner = launchTask(task, queryId, noName, jobname, jobs, driverCxt);
```
launchTask方法中新建了一个TaskRunner对象，并运行，如下：
```
TaskRunner tskRun = new TaskRunner(tsk, tskRes);
tskRun.runSequential();
```
TaskRunner的runSequential方法启动任务，并设置返回值。
```
  /**
   * Launches a task, and sets its exit value in the result variable.
   */
  public void runSequential() {
    int exitVal = -101;
    try {
      exitVal = tsk.executeTask();
    } catch (Throwable t) {
      if (tsk.getException() == null) {
        tsk.setException(t);
      }
      t.printStackTrace();
    }
    result.setExitVal(exitVal, tsk.getException());
  }
```
由此可见，在hiveServer启动任务时，便已报错，任务尚未发送到Hadoop，因此考虑调整HiveServer的内存大小。

## 解决办法
在`hive-env.sh`中设置`export HADOOP_HEAPSIZE=1024`  
或在`hadoop-env.sh`中设置`export HADOOP_CLIENT_OPTS="-Xmx1024m $HADOOP_CLIENT_OPTS"`
`注意，该设置会影响其他hadoop客户端命令`
