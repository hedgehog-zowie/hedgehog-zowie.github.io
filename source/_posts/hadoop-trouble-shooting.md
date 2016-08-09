---
title: hadoop-trouble-shooting
date: 2015-02-14 15:58:18
toc: true
tags: 
- hadoop hive
categories: 
- hadoop
---

此处记录工作过程中遇到的问题、原因及解决办法

# hadoop集群中一节点在任务运行过程中负载非常高

## 问题描述

在执行HIVE进行查询时发现，有一个节点（12核、32G内存）的负载达到了60多，且在该节点上运行的Map和Reduce都非常慢，使用top命令看到其wa cpu占用达到50%以上，有时候甚至100%。

## 原因

经一系列测试与观察发现，在该节点上进行读取的速度非常慢，而写入的速度与其他节点相差无几。
磁盘读写测试如下：

写：
```
[hadoop@bis-newdatanode-s2b-78 czw]$ time dd if=/dev/zero of=/data/czw/test.dbf bs=8k count=300000
300000+0 records in
300000+0 records out
2457600000 bytes (2.5 GB) copied, 2.94392 s, 835 MB/s

real    0m2.945s
user    0m0.040s
sys     0m2.904s
```
```
[hadoop@bis-newdatanode-s2b-77 czw]$ time dd if=/dev/zero of=/data/czw/test.dbf bs=8k count=300000
300000+0 records in
300000+0 records out
2457600000 bytes (2.5 GB) copied, 2.67449 s, 919 MB/s

real    0m2.697s
user    0m0.041s
sys     0m2.629s
```

读：
```
[hadoop@bis-newdatanode-s2b-78 czw]$ sudo time dd if=/dev/mapper/datavg-datalv of=/dev/null bs=8k
116913+0 records in
116912+0 records out
957743104 bytes (958 MB) copied, 108.427 s, 8.8 MB/s
0.01user 0.99system 1:48.42elapsed 0%CPU (0avgtext+0avgdata 3344maxresident)k
1381376inputs+0outputs (0major+243minor)pagefaults 0swaps
```
```
[hadoop@bis-newdatanode-s2b-77 czw]$ sudo time dd if=/dev/mapper/datavg-datalv of=/dev/null bs=8k
8624017+0 records in
8624016+0 records out
70647939072 bytes (71 GB) copied, 98.3302 s, 718 MB/s
0.94user 77.33system 1:38.34elapsed 79%CPU (0avgtext+0avgdata 3360maxresident)k
137980240inputs+0outputs (2major+242minor)pagefaults 0swaps
```

# Reduce ShuffleError: error in shuffle in fetcher, OutOfMemoryError

## 问题描述

发现一个job偶尔会出现ShuffleError错误，重新执行又可以正常运行，查看其错误日志为：
```
org.apache.hadoop.mapred.TaskAttemptListenerImpl: Task: attempt_1468643232136_15659_r_000044_0 - exited : org.apache.hadoop.mapreduce.task.reduce.Shuffle$ShuffleError: error in shuffle in fetcher#1
	at org.apache.hadoop.mapreduce.task.reduce.Shuffle.run(Shuffle.java:121)
	at org.apache.hadoop.mapred.ReduceTask.run(ReduceTask.java:380)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:162)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:415)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1491)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:157)
Caused by: java.lang.OutOfMemoryError: Java heap space
	at org.apache.hadoop.io.BoundedByteArrayOutputStream.<init>(BoundedByteArrayOutputStream.java:56)
	at org.apache.hadoop.io.BoundedByteArrayOutputStream.<init>(BoundedByteArrayOutputStream.java:46)
	at org.apache.hadoop.mapreduce.task.reduce.InMemoryMapOutput.<init>(InMemoryMapOutput.java:63)
	at org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl.unconditionalReserve(MergeManagerImpl.java:297)
	at org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl.reserve(MergeManagerImpl.java:287)
	at org.apache.hadoop.mapreduce.task.reduce.Fetcher.copyMapOutput(Fetcher.java:411)
	at org.apache.hadoop.mapreduce.task.reduce.Fetcher.copyFromHost(Fetcher.java:341)
	at org.apache.hadoop.mapreduce.task.reduce.Fetcher.run(Fetcher.java:165)
```

## 原因

我们的参数设置：
mapreduce.reduce.shuffle.parallelcopies=5`(reuduce shuffle阶段并行传输数据的Fetcher线程数)`
mapreduce.reduce.shuffle.input.buffer.percent=0.9`(copy阶段内存使用的最大值)`
mapreduce.reduce.shuffle.memory.limit.percent=0.25`(每个fetch取到的输出的大小能够占的内存比的大小)`

综上，实际每个fetcher的输出能放在内存的大小是：`reducer的java heap size * 0.9 * 0.25`。
若5个线程均需申请这么多内存，则需要的内存大小为：`reducer的java heap size * 0.9 * 0.25 * 5`，便超出了reducer的最大内存，从而报出`Java heap space OutOfMemoryError`。

`因此，我们可以采取调整上述3个参数的大小，控制它们的积不超过1即可`。

## 参考
[hadoop Shuffle Error OOM错误分析和解决](http://brandnewuser.iteye.com/blog/2149176)

