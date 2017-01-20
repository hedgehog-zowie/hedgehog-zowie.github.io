---
title: spark-trouble-shooting
date: 2016-10-14 17:37:51
toc: true
tags:
- spark
categories: 
- spark
---

记录使用Spark过程中遇到的问题、原因及解决办法。

# java.lang.IllegalArgumentException: Size exceeds Integer.MAX_VALUE

执行Spark任务时，抛出如下异常：

```
16/12/26 13:45:11 INFO client.AppClient$ClientEndpoint: Executor updated: app-20161216/12/26 13:46:39 INFO storage.BlockManagerInfo: Added rdd_14_0 on disk on bis-newdatanode-s2
b-75:54113 (size: 7.5 GB)
16/12/26 13:46:39 WARN scheduler.TaskSetManager: Lost task 0.0 in stage 3.0 (TID 6, bis-newdatanode-s2b-75): java.lang.IllegalArgumentException: Size exceeds Integer.MAX_VALUE
        at sun.nio.ch.FileChannelImpl.map(FileChannelImpl.java:789)
        at org.apache.spark.storage.DiskStore$$anonfun$getBytes$2.apply(DiskStore.scala:127)
        at org.apache.spark.storage.DiskStore$$anonfun$getBytes$2.apply(DiskStore.scala:115)
        at org.apache.spark.util.Utils$.tryWithSafeFinally(Utils.scala:1250)
        at org.apache.spark.storage.DiskStore.getBytes(DiskStore.scala:129)
        at org.apache.spark.storage.DiskStore.getBytes(DiskStore.scala:136)
        at org.apache.spark.storage.BlockManager.doGetLocal(BlockManager.scala:503)
        at org.apache.spark.storage.BlockManager.getLocal(BlockManager.scala:420)
        at org.apache.spark.storage.BlockManager.get(BlockManager.scala:625)
        at org.apache.spark.CacheManager.putInBlockManager(CacheManager.scala:154)
        at org.apache.spark.CacheManager.getOrCompute(CacheManager.scala:78)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:268)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:306)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:270)
```

原因：由于spark存在限制“2GB limit in spark for blocks”，详见：https://issues.apache.org/jira/browse/SPARK-1476，因此考虑rdd重新分区如下：`itemRdd.repartition(100)`，问题解决。


# java.lang.IllegalArgumentException: requirement failed

使用KMeans算法进行评估时，出现如下错误：

```
16/12/26 16:13:40 WARN scheduler.TaskSetManager: Lost task 0.0 in stage 50.0 (TID 2418, bis-newdatanode-s2b-59): java.lang.IllegalArgumentException: requirement failed
        at scala.Predef$.require(Predef.scala:221)
        at org.apache.spark.mllib.util.MLUtils$.fastSquaredDistance(MLUtils.scala:330)
        at org.apache.spark.mllib.clustering.KMeans$.fastSquaredDistance(KMeans.scala:595)
        at org.apache.spark.mllib.clustering.KMeans$$anonfun$findClosest$1.apply(KMeans.scala:569)
        at org.apache.spark.mllib.clustering.KMeans$$anonfun$findClosest$1.apply(KMeans.scala:563)
        at scala.collection.mutable.ArraySeq.foreach(ArraySeq.scala:73)
        at org.apache.spark.mllib.clustering.KMeans$.findClosest(KMeans.scala:563)
        at org.apache.spark.mllib.clustering.KMeans$.pointCost(KMeans.scala:586)
        at org.apache.spark.mllib.clustering.KMeansModel$$anonfun$computeCost$1.apply(KMeansModel.scala:88)
        at org.apache.spark.mllib.clustering.KMeansModel$$anonfun$computeCost$1.apply(KMeansModel.scala:88)
        at scala.collection.Iterator$$anon$11.next(Iterator.scala:328)
        at scala.collection.Iterator$class.foreach(Iterator.scala:727)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1157)
        at scala.collection.TraversableOnce$class.foldLeft(TraversableOnce.scala:144)
        at scala.collection.AbstractIterator.foldLeft(Iterator.scala:1157)
        at scala.collection.TraversableOnce$class.fold(TraversableOnce.scala:199)
        at scala.collection.AbstractIterator.fold(Iterator.scala:1157)
        at org.apache.spark.rdd.RDD$$anonfun$fold$1$$anonfun$19.apply(RDD.scala:1086)
        at org.apache.spark.rdd.RDD$$anonfun$fold$1$$anonfun$19.apply(RDD.scala:1086)
        at org.apache.spark.SparkContext$$anonfun$36.apply(SparkContext.scala:1951)
        at org.apache.spark.SparkContext$$anonfun$36.apply(SparkContext.scala:1951)
        at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
        at org.apache.spark.scheduler.Task.run(Task.scala:89)
        at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1110)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:603)
        at java.lang.Thread.run(Thread.java:722)
```

原因：检查代码发现是进行评估的数据特征数与训练模型不一致，从而产生这种错误，但`不可排除其他原因也产生该错误`，该错误提示为参数非法，因此需`检查自己的代码`。


# RDD分区不能均匀地分发到各个executor上

RDD分区不能均匀地分发到各个executor上，未截取运行日志和截图，在网上找到个类似的问题：[RDD Partitions not distributed evenly to executors](http://apache-spark-developers-list.1001551.n3.nabble.com/RDD-Partitions-not-distributed-evenly-to-executors-td16988.html)
未找到正式的解决办法，通过加大分区数量暂时规避了此问题。

# spark.ContextCleaner: Error cleaning broadcast

不明原因，未找到正式的解决办法，通过加大分区数量暂时规避了此问题。


