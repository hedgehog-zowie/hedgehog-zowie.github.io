---
title: spark-tips
date: 2016-09-23 08:50:52
toc: true
tags:
- spark
categories:
- spark
---
记录使用spark的过程中一些技巧、方法、注意事项。

# executor-cores/executor-memory对执行效率的影响
环境：1 master/4 workers（12核，32G内存，其中spark可用资源为12核，16G内存）
程序说明：对约3000万行游戏数据进行ALS模型训练，并对所有用户（约600万）预测200款游戏的评分，最后取出每个用户20款评分最高的游戏进行推荐。
效率测试过程：
1.executor-cores 12, executor-memory 16g
这种配置下，一共会启用4个executor，共使用48 cores, 64G内存。
开始时间：16/09/22 16:06:32
结束时间：16/09/22 18:05:01
耗时：7109s = 118.5m
2.executor-cores 3, executor-memory 4g
这种配置下，一共会启用16个executor，共使用48 cores, 64G内存。
开始时间：2016/9/22 23:00:02
结束时间：2016/9/23 0:50:11
耗时：6609s = 110.15m
3.executor-cores 1, executor-memory 2g, total-executor-cores 16
这种配置下，一共会启用16个executor，共使用16 cores, 32G内存，可以结余出资源供其他程序使用。
开始时间：2016/9/22 18:32:17
结束时间：2016/9/22 21:21:47
耗时：10170s = 169.5m

结论：
`1.使用所有资源，其效率比使用部分资源效率要高，但并不能成比例增长；`
`2.增加executor个数可稍微提升执行效率。`
`PS: 由于集群规模有限，所得到的信息也相对有限，因此，针对不同集群规模，需做相应的测试，不可直接套用此结论。`

# scala.MatchError, GenericRowWithSchema
在评估ALS模型时，需要将predictions中的内容全部取出来计算方差和标准差，而在提取每行内容时，遇到错误如下：
```
16/09/29 10:40:59 WARN scheduler.TaskSetManager: Lost task 2.0 in stage 103.0 (TID 492, bis-newdatanode-s2b-67): scala.MatchError: [0.0013,0.018407427] (of class org.apache.spark.sql.catalyst.expressions.GenericRowWithSchema)
```
提示说明类型不匹配，查看我的代码：
```
val predictions = model.transform(test).cache()
val mse = predictions.select("rating", "prediction").rdd.flatMap {
                case Row(rating: Double, prediction: Double) =>
				...
				...
}
```
进一步查看transform方法：
```
override def transform(dataset: DataFrame): DataFrame = {
    // Register a UDF for DataFrame, and then
    // create a new column named map(predictionCol) by running the predict UDF.
    val predict = udf { (userFeatures: Seq[Float], itemFeatures: Seq[Float]) =>
      if (userFeatures != null && itemFeatures != null) {
        blas.sdot(rank, userFeatures.toArray, 1, itemFeatures.toArray, 1)
      } else {
        Float.NaN
      }
    }
    dataset
      .join(userFactors, dataset($(userCol)) === userFactors("id"), "left")
      .join(itemFactors, dataset($(itemCol)) === itemFactors("id"), "left")
      .select(dataset("*"),
        predict(userFactors("features"), itemFactors("features")).as($(predictionCol)))
  }
```
说明rating字段类型，与传入参数test中的rating字段类型一致，而prediction字段类型，是由blas.sdot()函数决定的，查看该函数： 
```
abstract public float sdot(int n, float[] sx, int incx, float[] sy, int incy);
```
其返回值是float型，因此，将提取行数据的代码修改为：
```
case Row(rating: Double, prediction: Float) =>
```
