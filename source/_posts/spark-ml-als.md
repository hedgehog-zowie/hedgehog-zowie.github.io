---
title: ALS矩阵分解算法
date: 2016-09-23 08:50:52
toc: true
tags:
- als spark ml
categories:
- spasrk
---

# 阿基米德项目ALS矩阵分解算法应用案例

原文地址：

[阿基米德项目ALS矩阵分解算法应用案例](https://github.com/ceys/jdml/wiki/ALS)

[格式比较好的版本](http://www.cnblogs.com/lengyue365/p/5689219.html)

## 算法描述

### 原理

#### 问题描述

ALS的矩阵分解算法常应用于推荐系统中，将用户(user)对商品(item)的评分矩阵，分解为用户对商品隐含特征的偏好矩阵，和商品在隐含特征上的映射矩阵。与传统的矩阵分解SVD方法来分解矩阵R($R\in \mathbb{R}^{m\times n}$)不同的是，ALS(alternating least squares)希望找到两个低维矩阵，以 $\tilde{R} = XY$ 来逼近矩阵R，其中 ，$X\in \mathbb{R}^{m\times d}$，$Y\in \mathbb{R}^{d\times n}$，d 表示降维后的维度，一般 d<<r，r表示矩阵 R 的秩，$r<<min(m,n)$。

#### 目标函数

* 为了找到低维矩阵X,Y最大程度地逼近矩分矩阵R，最小化下面的平方误差损失函数。
  `$$L(X,Y) = \sum_{u,i}(r_{ui} - x_{u}^{T}y_{i})^{2}......(1)$$`

* 为防止过拟合给公式 (1) 加上正则项，公式改下为： `$$L(X,Y) = \sum_{u,i}(r_{ui} - x_{u}^{T}y_{i})^{2} + \lambda (\left | x_{u}\right |^{2} +　\left | y_{i}\right |^{2})......(2)$$`
  其中$x_{u}\in \mathbb{R}^{d}，y_{i}\in \mathbb{R}^{d}$，$1\leqslant u\leqslant m$，$1\leqslant i\leqslant n$，$\lambda$是正则项的系数。

#### 模型求解

* 固定Y，对$x_{u}$ 求导 $\frac{\partial L(X,Y)}{\partial x_{u}} = 0$，得到求解$x_{u}$的公式
  `$$x_{u} = (Y^{T}Y + \lambda I )^{-1}Y^{T}r(u)......(3)$$`
* 同理固定X,可得到求解$y_{i}$的公式
  `$$y_{i} = (X^{T}X + \lambda I )^{-1}X^{T}r(i)......(4)$$`
  其中，$r_{u}\in \mathbb{R}^{n}$,$r_{i}\in \mathbb{R}^{m}$,I表示一个d * d的单位矩阵。
* 基于公式(3)、(4)，首先随机初始化矩阵X，然后利用公式(3)更新Y，接着用公式(4)更新X，直到计算出的RMSE(均方根误差)值收敛或迭代次数足够多而结束迭代为止。
  其中，$\tilde{R} = XY$，$RMSE = \sqrt{\frac{\sum (R - \tilde{R})^{2}}{N}}$

#### ALS-WR模型

以上模型适用于用户对商品的有明确的评分矩阵的场景，然而很多情况下用户没有明确的反馈对商品的偏好，而是通过一些行为隐式的反馈。比如对商品的购买次数、对电视节目收看的次数或者时长，这时我们可以推测次数越多，看得时间越长，用户的偏好程度越高，但是对于没有购买或者收看的节目，可能是由于用户不知道有该商品，或者没有途径获取该商品，我们不能确定的推测用户不喜欢该商品。ALS-WR通过置信度的权重来解决此问题，对于我们更确信用户偏好的项赋予较大的权重，对于没有反馈的项，赋予较小的权重。模型如下

* ALS-WR目标函数
  `$\underset{x_{u},y_{i}}{min} L(X,Y) = \sum_{u,i}c_{ui}(p_{ui} - x_{u}^{T}y_{i})^{2} + \lambda (\left | x_{u}\right |^{2} +　\left | y_{i}\right |^{2})......(5)$
  其中 $$p_{ui} = \begin{cases} & \text{1 if } r_{ui} > 0 \ & \text{0 if } r_{ui} = 0 \end{cases}$$
  $c_{ui} = 1 + \alpha r_{ui}$，$\alpha$是置信度系数
* 通过最小二乘法求解
  $$x_{u} = (Y^{T}C^{u}Y + \lambda I )^{-1}Y^{T}C^{u}r(u)......(6)$$
  $$y_{i} = (X^{T}C^{i}X + \lambda I )^{-1}X^{T}C^{i}r(i)......(7)$$
  其中$C^{u}$是一$n\times n$维的个对角矩阵，$C_{ii}^{u} = c_{ui}$; 其中$C^{u}$是一$m\times m$维的个对角矩阵，$C_{ii}^{u} = c_{ui}$

#### 与其他矩阵分解算法的比较

* 在实际应用中，由于待分解的矩阵常常是非常稀疏的，与SVD相比，ALS能有效的解决过拟合问题。
* 基于ALS的矩阵分解的协同过滤算法的可扩展性也优于SVD。
* 与随机梯度下降的求解方式相比，一般情况下随机梯度下降比ALS速度快；但有两种情况ALS更优于随机梯度下降：1)当系统能够并行化时，ALS的扩展性优于随机梯度下降法。2）ALS-WR能够有效的处理用户对商品的隐式反馈的数据。

### 伪代码

```
import numpy

def mf_als(R, P, Q, K, steps=5000, alpha=0.0002, beta=0.02):
    Q = Q.T
    for step in xrange(steps): 
        for i in xrange(len(P)):                              
            e = 0
        for i in xrange(len(R)):
            for j in xrange(len(R[i])):
                if R[i][j] > 0:
                    e = e + pow(R[i][j] - numpy.dot(P[i,:],Q[:,j]), 2)
                    for k in xrange(K):
                        e = e + (beta/2) * (pow(P[i][k],2) + pow(Q[k][j],2))
        if e < 0.001:
        break
    return P, Q.T

if __name__ == "__main__":
    R = [
     [5,3,0,1],
     [4,0,0,1],
     [1,1,0,5],
     [1,0,0,4],
     [0,1,5,4],
    ]
    R = numpy.array(R)
    N, M, K = len(R), len(R[0]), 2
    P = numpy.random.rand(N,K)
    Q = numpy.random.rand(M,K)

    nP, nQ = mf_als(R, P, Q, K)
    print numpy.dot(nP, nQ.T)
```

### 并行化方法

整体思路就是把矩阵拆成行向量，分别来做最小二乘参数估计。

伪代码中，所有数据都被广播到了集群节点。实际代码中，只会向各节点分发其运算能用到的部分数据。

```
# M： item个数， U： user个数， F： 分解矩阵的秩
# 初始化评分矩阵
R = matrix(rand(M, F)) * matrix(rand(U, F).T)
ms = matrix(rand(M ,F))
us = matrix(rand(U, F))

# 将评分矩阵，item矩阵，user矩阵广播到所有节点
Rb = sc.broadcast(R)
msb = sc.broadcast(ms)
usb = sc.broadcast(us)

# 指定遍历次数ITERATIONS
for i in range(ITERATIONS):
    # 固定user矩阵，分布式求解item矩阵
    # 每个节点计算M/slices个items
    ms = sc.parallelize(range(M), slices) \
           .map(lambda x: update(x, msb.value[x, :], usb.value, Rb.value)) \
           .collect()
    ms = matrix(np.array(ms)[:, :, 0])      # collect() returns a list, so array ends up being
                                            # a 3-d array, we take the first 2 dims for the matrix
    # 广播更新后的item矩阵
    msb = sc.broadcast(ms)

    # 固定item矩阵，分布式求解user矩阵
    us = sc.parallelize(range(U), slices) \
           .map(lambda x: update(x, usb.value[x, :], msb.value, Rb.value.T)) \
           .collect()
    us = matrix(np.array(us)[:, :, 0])
    usb = sc.broadcast(us)

    # 平方误差
    error = rmse(R, ms, us)

# 最小二乘更新数据
# 输入：矩阵行index，要更新的特征向量，固定的特征矩阵，评分矩阵
def update(i, vec, mat, ratings):
    uu = mat.shape[0]
    ff = mat.shape[1]

    # 变成可逆矩阵
    XtX = mat.T * mat
    Xty = mat.T * ratings[i, :].T

    # 正则项
    for j in range(ff):
        XtX[j,j] += LAMBDA * uu

    # XtXZ=XtY，求Z并返回
    # 返回类型为二维数组。因为每次update只计算一个向量，所以实际只有第一维有值。
    return np.linalg.solve(XtX, Xty)
```

### 文献

* [Large-scale Parallel Collaborative Filtering for the Netfli Prize](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.173.2797&rep=rep1&type=pdf)

* [Collaborative Filtering for Implicit Feedback Datasets](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=4781121)

* [MATRIX FACTORIZATION TECHNIQUES FOR RECOMMENDER SYSTEMS](http://rakaposhi.eas.asu.edu/cse494/lsi-for-collab-filtering.pdf)

## 具体实现及调用

### 模型调用

**输入数据结构与说明**：

`Rating(UserId:Int, ItemId:Int, Rating:toDouble)` 用户、商品id必须为整形，评分为浮点型。

**模型输出数据结构及说明**：

`RDD[(Id:Int, Array[feature:Double]]]` 可以分别输出userFeatures和itemFeatures。包含id和隐含特征值。

**推荐结果输出数据结构及说明**：
`Rating(UserId:Int, ItemId:Int, Rating:toDouble)` 用户、商品id与预测评分。

**算法调用语句示例**：

```
import org.apache.spark.mllib.recommendation.ALS
import org.apache.spark.mllib.recommendation.Rating

// Load and parse the data
val data = sc.textFile("mllib/data/als/test.data")
val ratings = data.map(_.split(',') match {
    case Array(user, item, rate) =>  Rating(user.toInt, item.toInt, rate.toDouble)
})

// Build the recommendation model using ALS
val numIterations = 20
val model = ALS.train(ratings, 1, 20, 0.01)

// Evaluate the model on rating data
val usersProducts = ratings.map{ case Rating(user, product, rate)  => (user, product)}
val predictions = model.predict(usersProducts).map{
    case Rating(user, product, rate) => ((user, product), rate)
}
val ratesAndPreds = ratings.map{
    case Rating(user, product, rate) => ((user, product), rate)
}.join(predictions)
val MSE = ratesAndPreds.map{
    case ((user, product), (r1, r2)) =>  math.pow((r1- r2), 2)
}.reduce(_ + _)/ratesAndPreds.count
println("Mean Squared Error = " + MSE)
```

**性能参数配置**：

```
val conf = new SparkConf()
  .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
  .set("spark.kryo.registrator",  classOf[ALSRegistrator].getName)
    /*
    Whether to track references to the same object when serializing data with Kryo, 
    which is necessary if your object graphs have loops 
    and useful for efficiency if they contain multiple copies of the same object. 
    Can be disabled to improve performance if you know this is not the case.
    */
  .set("spark.kryo.referenceTracking", "false")
  .set("spark.kryoserializer.buffer.mb", "8")
    /*
    Number of milliseconds to wait to launch a data-local task before giving up and launching it on a less-local node. 
    You should increase this setting if your tasks are long and see poor locality, but the default usually works well.
    */
  .set("spark.locality.wait", "10000")
```

val conf = new SparkConf()
  .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
  .set("spark.kryo.registrator",  classOf[ALSRegistrator].getName)
    /*
    Whether to track references to the same object when serializing data with Kryo, 
    which is necessary if your object graphs have loops 
    and useful for efficiency if they contain multiple copies of the same object. 
    Can be disabled to improve performance if you know this is not the case.
    */
  .set("spark.kryo.referenceTracking", "false")
  .set("spark.kryoserializer.buffer.mb", "8")
    /*
    Number of milliseconds to wait to launch a data-local task before giving up and launching it on a less-local node. 
    You should increase this setting if your tasks are long and see poor locality, but the default usually works well.
    */
  .set("spark.locality.wait", "10000")

## 案例描述

### 业务问题描述及分析

#### 问题描述

在电子商务领域中，当用户面对大量的商品时，往往无法快速找到自己喜欢的商品，或者不是非常明确的知道自己喜欢商品。和搜索引擎相比的推荐系统通过研究用户的兴趣偏好，进行个性化计算，由系统发现用户的兴趣点，从而引导用户发现自己的需求。

#### 简要分析

矩阵分解是推荐系统中非常重要的一种算法，它通过将用户对商品的评分矩阵（或者隐含数据），分解为用户对商品隐含特征的偏好矩阵，和商品在隐含特征上的映射矩阵。如果用户所偏好特征，在商品上基本都出现，我们可以认为这个商品是用户喜欢的，进而可以将该商品推荐给用户。

我们用历史的订单数据作为训练数据，来预测用户对未购买过的商品的偏好程度，将偏好程度最高topN的商品推荐给用户。

### 数据的准备

图书品类下，2014年1月到5月的订单数据，取在1~4月和4~5月两个区间都有图书购物记录的用户。1~4月为训练数据，4~5月为测试数据。用户对商品有购买行为，则隐性反馈值为1。

### 算法的运行及模型生成

#### 性能

1. `N = User*Item` N的最大值（理论估计+实际验证） 测试了两组数据集：

第一组：

* 训练： pair：6557620 用户：781030 商品：726490

* 测试： pair：3250426 用户：781030 商品：490257

```
N = 726490*781030 = 567410484700

稀疏度 =pair/N = 0.0000115571
```

1. `worker-num,worker-mem,blocks,kryo,kryo-reference,locality-wait` 等运行参数与数据量对一轮迭代时间的影响。
2. 运行时rdd的transform和action的运算时间与shuffle大小。

#### 模型

数据性质：

    稀疏性（行为（评分、购买），品类）

参数选择：

	lambda，alpha，R，iter

### 模型的评估

**矩阵分解的评估**

* 原始矩阵为R，预测的为$\tilde{R} = U^{T}V$，用RMSE来评估预测的效果。
  $$RMSE = \sqrt{\frac{\sum (R - \tilde{R})^{2}}{N}}$$
  其中N为中所有求和的项数

**推荐效果的评估**

* 对推荐预测的效果一般用准确率(precision)和召回率(recall)来衡量。R(u)是根据用户在训练集上的行为给用户推荐的列表，T(u)是用户在测试集上的行为列表。则有
  
  召回率
  $$Recall = \frac{\sum_{u\in U }\left |R(u)\bigcap T(u) \right |}{\sum_{u\in U }\left |T(u) \right |}$$
  
  准确率
  $$Precise = \frac{\sum_{u\in U }\left |R(u)\bigcap T(u) \right |}{\sum_{u\in U }\left |R(u) \right |}$$

## 与mahout的对比

mahout与spark性能对比

 * 数据量 6991409行，134M
 * 集群环境：mahout与spark安装在同一集群环境
 * 影响运行时间的参数：降维后的秩 30，迭代次数 30，mahout与spark设置相同
 * 运行时间：mahout(10个reduce) 运行180 minutes，spark 运行 40 minutes

