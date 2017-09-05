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

# spark thrift server执行hive sql遇到Filesystem closed异常

```
17/05/03 17:34:05 WARN scheduler.TaskSetManager: Lost task 9.0 in stage 28962.0 (TID 12380244, bis-newdatanode-s2a-22): org.apache.hadoop.hive.ql.metadata.HiveException: java.io
.IOException: Filesystem closed
        at org.apache.hadoop.hive.ql.io.HiveFileFormatUtils.getHiveRecordWriter(HiveFileFormatUtils.java:249)
        at org.apache.spark.sql.hive.SparkHiveWriterContainer.initWriters(hiveWriterContainers.scala:114)
        at org.apache.spark.sql.hive.SparkHiveWriterContainer.executorSideSetup(hiveWriterContainers.scala:86)
        at org.apache.spark.sql.hive.execution.InsertIntoHiveTable.org$apache$spark$sql$hive$execution$InsertIntoHiveTable$$writeToFile$1(InsertIntoHiveTable.scala:102)
        at org.apache.spark.sql.hive.execution.InsertIntoHiveTable$$anonfun$saveAsHiveFile$3.apply(InsertIntoHiveTable.scala:84)
        at org.apache.spark.sql.hive.execution.InsertIntoHiveTable$$anonfun$saveAsHiveFile$3.apply(InsertIntoHiveTable.scala:84)
        at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
        at org.apache.spark.scheduler.Task.run(Task.scala:89)
        at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1110)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:603)
        at java.lang.Thread.run(Thread.java:722)
Caused by: java.io.IOException: Filesystem closed
        at org.apache.hadoop.hdfs.DFSClient.checkOpen(DFSClient.java:629)
        at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1365)
        at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1307)
        at org.apache.hadoop.hdfs.DistributedFileSystem$6.doCall(DistributedFileSystem.java:384)
        at org.apache.hadoop.hdfs.DistributedFileSystem$6.doCall(DistributedFileSystem.java:380)
        at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
        at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:380)
        at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:324)
        at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:905)
        at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:798)
        at org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat.getHiveRecordWriter(HiveIgnoreKeyTextOutputFormat.java:80)
        at org.apache.hadoop.hive.ql.io.HiveFileFormatUtils.getRecordWriter(HiveFileFormatUtils.java:261)
        at org.apache.hadoop.hive.ql.io.HiveFileFormatUtils.getHiveRecordWriter(HiveFileFormatUtils.java:246)
```

原因：HDFS故障重启后未重启spark thriftserver

# spark task未序列化

异常信息如下：

```
spark task未序列化问题：
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
        at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:304)
        at org.apache.spark.util.ClosureCleaner$.org$apache$spark$util$ClosureCleaner$$clean(ClosureCleaner.scala:294)
        at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:122)
        at org.apache.spark.SparkContext.clean(SparkContext.scala:2055)
        at org.apache.spark.rdd.RDD$$anonfun$mapPartitions$1.apply(RDD.scala:707)
        at org.apache.spark.rdd.RDD$$anonfun$mapPartitions$1.apply(RDD.scala:706)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:150)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:111)
        at org.apache.spark.rdd.RDD.withScope(RDD.scala:316)
        at org.apache.spark.rdd.RDD.mapPartitions(RDD.scala:706)
        at org.apache.spark.sql.DataFrame.mapPartitions(DataFrame.scala:1426)
        at com.gionee.gdata.recommender.kmeans.Recommend.recommend(Recommend.scala:343)
        at com.gionee.gdata.recommender.kmeans.Recommend.recommend(Recommend.scala:310)
        at com.gionee.gdata.recommender.kmeans.TrainAndRecommend$.main(TrainAndRecommend.scala:63)
        at com.gionee.gdata.recommender.kmeans.TrainAndRecommend.main(TrainAndRecommend.scala)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:601)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:731)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:181)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:206)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:121)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.io.NotSerializableException: com.gionee.gdata.recommender.kmeans.Recommend
Serialization stack:
        - object not serializable (class: com.gionee.gdata.recommender.kmeans.Recommend, value: com.gionee.gdata.recommender.kmeans.Recommend@3a85b804)
        - field (class: com.gionee.gdata.recommender.kmeans.Recommend$$anonfun$recommend$2, name: $outer, type: class com.gionee.gdata.recommender.kmeans.Recommend)
        - object (class com.gionee.gdata.recommender.kmeans.Recommend$$anonfun$recommend$2, <function1>)
        at org.apache.spark.serializer.SerializationDebugger$.improveException(SerializationDebugger.scala:40)
        at org.apache.spark.serializer.JavaSerializationStream.writeObject(JavaSerializer.scala:47)
        at org.apache.spark.serializer.JavaSerializerInstance.serialize(JavaSerializer.scala:101)
        at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:301)
        ... 23 more
```

解决办法：使用局部变量。
[参考链接](http://blog.csdn.net/sogerno1/article/details/45935159)

# file exists and does not match contents

spark thrift server执行hive sql时遇到文件存在但不匹配的错误，详细日志如下：
```
Error: org.apache.spark.SparkException: Job aborted due to stage failure: Task 97 in stage 55603.0 failed 4 times, most recent failure: Lost task 97.3 in stage 55603.0 (TID 10938135, bis-hadoop-datanode-s2d-135): org.apache.spark.SparkException: File ./hiveudf2.jar exists and does not match contents of http://10.10.10.80:54525/jars/hiveudf2.jar
```
日志提示hiveudf2.jar存在，但两个executor中的hiveudf2.jar文件不一致，从而导致该语句无法执行，基本会引起其他一些毫不相关（没有使用到hiveudf2.jar）的语句也无法执行；
检查work目录下的hiveudf2.jar文件：

```
[hadoop@bis-hadoop-datanode-s2d-135 spark-1.6.1-bin-2.2.0]$ find work/ -name hiveudf2.jar | xargs ls -l
-rwxrwxr-x 1 hadoop hadoop 18482684 May 31 04:31 work/app-20170518113247-0001/145/hiveudf2.jar
-rwxrwxr-x 1 hadoop hadoop 18482684 Jun  2 02:41 work/app-20170518113247-0001/149/hiveudf2.jar
-rwxrwxr-x 1 hadoop hadoop  1441792 Jun  5 05:24 work/app-20170518113247-0001/153/hiveudf2.jar
-rwxrwxr-x 1 hadoop hadoop 18482684 May 18 14:16 work/app-20170518113247-0001/4/hiveudf2.jar
-rwxrwxr-x 1 hadoop hadoop 18482684 May 18 14:16 work/app-20170518113247-0001/5/hiveudf2.jar
-rwxrwxr-x 1 hadoop hadoop 18482684 May 18 14:16 work/app-20170518113247-0001/6/hiveudf2.jar
-rwxrwxr-x 1 hadoop hadoop 18482684 May 18 14:16 work/app-20170518113247-0001/7/hiveudf2.jar
```
发现存在大小不一致的jar包，尝试打开work/app-20170518113247-0001/153/hiveudf2.jar，出现错误，因此，可能是在复制该JAR包时，出现错误导致JAR包不完整。

解决办法：
不再使用临时函数，全部使用全局函数，由于spark thrift server重启之后所有新添加的函数都会消失，因此需要在每次重启之后，添加这些全局函数；

`更好的解决办法：修改spark源码，增加这些函数，重新打包。`

[参考链接](http://coolplayer.net/2016/10/12/spark-%E6%8A%A5%E9%94%99%E5%AE%9A%E4%BD%8D/)

# 两个toArray直接相加出错Error:scalac: type mismatch

自己使用scala写的程序编译时出现错误：

```
Error:scalac: type mismatch;
 found   : Array[?B]
 required: scala.collection.GenTraversableOnce[?]
Note that implicit conversions are not applicable because they are ambiguous:
 both method booleanArrayOps in object Predef of type (xs: Array[Boolean])scala.collection.mutable.ArrayOps[Boolean]
 and method byteArrayOps in object Predef of type (xs: Array[Byte])scala.collection.mutable.ArrayOps[Byte]
 are possible conversion functions from Array[?B] to scala.collection.GenTraversableOnce[?]
```

检查代码发现是由这样一行代码引起：

```
(item, Vectors.dense(valueFeatureVector.toArray ++ normalizedValueFeatureVector.toArray))
```

将代码改成

```
val featureVectorArray  = valueFeatureVector.toArray ++ normalizedValueFeatureVector.toArray
(item, Vectors.dense(featureVectorArray))
```
依旧出错：

```
Error:(78, 86) polymorphic expression cannot be instantiated to expected type;
 found   : [B >: Double]Array[B]
 required: scala.collection.GenTraversableOnce[?]
```

因此可以确定是由于两个toArray返回的Array类型无法推断为一致所致，将代码继续改写如下：

```
val valueFeatureVectorArray:Array[Double] = valueFeatureVector.toArray
val normalizedValueFeatureVectorArray:Array[Double] = normalizedValueFeatureVector.toArray
val featureVectorArray  = valueFeatureVectorArray ++ normalizedValueFeatureVectorArray
```

编译成功

# spark maxResultSize的限制

```
Exception in thread "main" org.apache.spark.SparkException: Job aborted due to stage failure: Total size of serialized results of 102 tasks (1031.7 MB) is bigger than spark.driver.maxResultSize (1024.0 MB)
```

spark action的所有序列化结果的总大限制默认为1024M，若超出该值，则程序会终止，设置为0表示无限制。

[参考资料：Spark配置参数](http://www.tuicool.com/articles/zIvayyf) 

# spark程序编译异常

使用maven打包spark程序编译异常，报出如下错误：

```
[ERROR] Failed to execute goal org.scala-tools:maven-scala-plugin:2.15.2:compile (scala-compile-firs
t) on project ad-ctr-api: wrap: org.apache.commons.exec.ExecuteException: Process exited with an err
or: 1(Exit value: 1) -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following arti
cles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
```

同时有另一项目，pom.xml文件中指定了不同版本的参数，直接使用命令`mvn clean package`，其使用的scala版本是2.11，可以打包；但使用命令`mvn -Dscala-2.10 clean package`（scala-2.10是自定义的参数），打包失败，也报出上述相同错误。

解决办法：
`综上，可以确定是由于scala版本引起的原因，pom.xml中指定使用2.10版本的scala，开发时也是使用的2.10版本，而本机环境变量中配置的scala版本却为2.11，将其配置成2.10即可。`

`java版本不一致也会引起上述问题。`
