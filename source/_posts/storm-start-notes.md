---
title: storm-started-notes
date: 2017-06-26 00:00:00
toc: true
tags:
- storm
categories:
- storm
---

《Getting Started With Storm》读书笔记。

# 第一章：基础

* storm是一个分布式的，可靠的，容错的数据流处理系统。
* spout是storm集群中负责处理输入流的组件，spout将数据传输给bolt。
* bolt是storm集群中负责数据转化(原文是transforms)的组件，除此之外，它还可以将数据持久化存储，或者将数据再传递给其他的bolt。
* storm集群由一个master node和若干个work node组成；master node上有一个名为nimbus的守护进程，负责分发代码到集群上的各节点，为各work node分配任务，并进行监控；work node上有一个名为supervisor的守护进程，执行一个拓扑（topology）上的一部分；storm上的一个topology由分布于不同机器上的许多work node组成。
* storm将集群状态保存在zookeeper或本地磁盘上，当守护进程状态丢失、失败或重启时，不会影响到整个集群的健康。
* storm使用zeromq([0mq](http://zeromq.org/))来实现底层的网络通信。

# 第二章：开始

* storm有local和remote两个模式。在local模式下，storm topology在单个jvm上执行；remote模式下，我们提交一个一个topology到集群上运行。在local模式下运行一个topology与在集群上运行是相似的，需要注意的是，所有的组件之间是线程安全的，因为在集群模式下，它们运行于不同的JVM甚至不同的物理机上，这些JVM或物理机之间没有共享内存或直接联系。 正式部署于生产环境之前，在单机上使用remote模式是一个好主意。
* Hello World的例子 -- 统计词频，流程图如下：
![HelloWorld](chapter2-hello-world.png)
示例是其于storm 0.71版本的，修改为当前稳定版本1.1.0时，需要修改pom文件中的依赖，如下：
```
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>storm-core</artifactId>
    <version>1.1.0</version>
</dependency>
```
还需要修改各个类中的import包名。

TopologyMain是该程序的入口，其主要作用是定义Topology，Config，创建LocalCluster，并提交任务。
`注意在TopologyMain类中有一句Thread.sleep(1000);这个时间太短，容易造成zookeeper抛出ServerCnxn$EndOfStreamException异常，详细错误如下：`

```
org.apache.storm.shade.org.apache.zookeeper.server.ServerCnxn$EndOfStreamException: Unable to read additional data from client sessionid 0x15ce79acdd8000a, likely client has closed socket
	at org.apache.storm.shade.org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:228) [storm-core-1.1.0.jar:1.1.0]
	at org.apache.storm.shade.org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:208) [storm-core-1.1.0.jar:1.1.0]
	at java.lang.Thread.run(Thread.java:745) [?:1.8.0_91]
```

WordReader是该程序的spout，它读取文本文件的内容，并将其传递给bolts；open方法是第一个被调用的方法。

WordNormalizer和WordCounter是bolts，bolt每接收到一个元组都会执行execute，处理完数据之后将其进行存储或再发送到其他bolt。

运行结果：

```
-- Word Counter [word-counter-2] --
but: 1
very: 1
storm: 3
test: 1
application: 1
are: 1
is: 2
simple: 1
powerfull: 1
great: 2
an: 1
really: 1
```

# 第三章：Topologies

## Stream Grouping

* 设计Topology最重要的一件事情是定义数据在组件之间的交换（即数据流被bolt消费），Stream Grouping指定bolt消费哪些数据流，以及如何消费它们。
* 一个节点能够提交多个数据流，一个Stream Grouping允许我们选择接收哪个数据流。
* Shuffle Grouping是最常用的grouping，它只有一个参数 —— 源组件ID（如第二章示例的"word-reader"）。只适用于原子操作，如果数据不能被随机分布，则不适用。
* Fields Grouping允许通过一个或多个Fields来控制tuples发送给bolts的方式，将Fields组合相同的tuples发送到同一个bolt，如第二章的示例：
```
builder.setBolt("word-counter", new WordCounter(),2)
.fieldsGrouping("word-normalizer", new Fields("word"));
```
* All Grouping发送每一个元组的拷贝到各个接收bolt的实例，这种方式用于向bolts发送信号。
* Custom Grouping，可以通过实现org.apache.storm.grouping.CustomStreamGrouping接口来自定义grouping，通过这种方式来决定哪些bolts接收哪些tuples。
* Direct Grouping由source决定哪个组件接收tuple，使用emitDirect方法代替emit。
* Global Grouping将所有数据源产生的所有tuples发送到单一实例。
* None Grouping相当于随机数据流组。

## LocalCluster和StormSubmitter比较

* LocalCluster用于本地运行topology，StormSubmitter用于向真实集群提交topology，如`StormSubmitter.submitTopology("Count-Word-Topology-With_Refresh-Cache", conf, builder.createTopology());`。
* 打好jar包之后，使用命令`storm jar allmycode.jar org.me.MyTopology arg1 arg2 arg3`提交任务，如`storm jar target/Topologies-0.0.1-SNAPSHOT.jar countword.TopologyMain src/main/resources/words.txt`；使用命令`storm kill`杀死任务，如`storm kill Count-Word-Topology-With-Refresh-Cache`。
* `注意：topology的名称必须唯一。`

##　DRPC Topologies

* DRPC是Distributed Remote Procedure Call的缩写，译为“分布式远程过程调用”。
![DRPC](chapter3-drpc.png)
* DRPC server是client和storm topology之间的连接器，作为topology spout的数据源，接收一个待执行的函数和它的参数。对函数操作的每一片数据，DRPC server会为每一个RPC分配一个请求ID，在topology执行完最后一个bolt时，必须发送RPC请求ID和结果，DRPC server再将结果返回给client。
`一个DRPC可以执行多个function，每个function的名称必须不同。`
* LinearDRPCTopologyBuilder是构造DRPC的工具，1.1.0版本已不推荐使用(Trident subsumes the functionality provided by this class, so it's deprecated)。
* 使用DRPCClient连接到远程的DRPC server。

# 第四章：Spouts

* storm需要开发者来保证消息的可靠性。
* 在spout中管理可靠性，需要在emit tuple的时候包含一个消息ID，如`collector.emit(new Values(…),tupleId)`，当一个tuple被正确处理时调用ack方法，处理失败时调用fail方法；当所有的bolt处理成功后，tuple才算处理成功。

# 第五章：Bolts

* bolt的生命周期：bolt由client创建，序列化为topology，然后提交到集群的master；master启动workers反序列化bolt，调用prepare，然后开始处理tuple。
* bolt的结构：
```
declareOutputFields(OutputFieldsDeclarer declarer)
    为bolt声明输出模式
prepare(java.util.Map stormConf, TopologyContext context, OutputCollector collector)
    仅在bolt开始处理元组之前调用
execute(Tuple input)
    处理输入的单个元组
cleanup()
    在bolt即将关闭时调用
```
* 同spout一样，bolt也需要开发者来保证消息的可靠性，collector.ack(tuple)和collector.fail(tuple)告诉spout每条消息发生了什么。当树上的所有消息都被处理完了，storm才认为spout的一个tuple处理完了，若未在配置的时间（默认值是30秒，可以通过修改Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS来修改超时时间）内完成，也认为是失败。
* 每一个tuple必须是acked或failed，storm使用内存跟踪每个tuple，因此，如果不对每个tuple调用ack或fail，任务会耗尽内存。
* 使用emit(streamId, tuple)方法，一个bolt可以将tuple发送到多个数据流，streamId是一个标识流的字符串，在TopologyBuilder中，可以决定哪个流来订阅它。
* 在join和aggregate流中，需要在内存中缓存tuples，在这种场景下为了确保消息完成，需要将数据流锚定到多个tuple，实现方式如下：
```
List<Tuple> anchors = new ArrayList<Tuple>();
anchors.add(tuple1);
anchors.add(tuple2);
collector.emit(anchors, values);
```
通过这种方式，任何一个bolt调用acks或fails，都会通知给tree，因为流锚定到多个tuple，所有调用的spout都会被通知。
* IBasicBolt接口在execute执行完成之后，自行调用ack方法，BaseBasicBolt是它的一个实现类，发送给BasicOutputCollector的tuples会自动锚定到输入tuple。

# 第六章：实例



# 第七章：使用非JVM语言

* 由于storm topology是Thrift结构的，Nimbus是一个Thrift进程，因此，可以使用任何语言创建和提交任务。
* ShellSpout和ShellBolt用于执行和控制由其他语言编写的spout和bolt；

##　多语言协议说明

1. 初始化握手
2. 开始循环
3. 读/写tuples

### 初始化握手

* 为了控制进程（start和stop），storm需要知道所执行脚本的PID。根据协议，storm首先需要发送一个包含storm config, topology context和pid目录的JSON对象作为输入；看起来是这样的：
```
{
  "conf": {
    "topology.message.timeout.secs": 3,
    // etc
  },
  "context": {
    "task->component": {
      "1": "example-spout",
      "2": "__acker",
      "3": "example-bolt"
    },
    "taskid": 3
  },
  "pidDir": "..."
}
```
脚本进程必须在pidDir下创建一个空文件，以进程号作为文件名，内容为：
```
{"pid": 1234}
```
storm对这个PID进行跟踪，并在停止的时候关闭它。

### 开始循环和读/写tuples

* 这一步的实现取决于它是spout还是bolt。如果是spout，需要发送tuples；如果是bolt，需要循环并且读取tuples，处理它们，继续发送tuples到其他bolt，并调用ack,fail；

# 第八章：事务型topologies

* 在事务型topology中，storm使用并行和顺序的方式提供tuples。spout并行生成供bolts处理的tuple批次；一些bolts按严格的顺序提交它们的处理批次。
* 使用TransactionalTopologyBuilder构建topology。
* 继承BaseTransactionalSpout接口实现spout；继承ITransactionalSpout.Coordinator接口实现Coordinator（协调者）；继承ITransactionalSpout.Emitter接口实现Emitter（分发器）。
* 事务型topology可以使用IBasicBolt,BaseBatchBolt和Committer Bolts；BaseBatchBolt的execute方法会处理接收到的tuple，但不会再分发新的tuple，批次处理完成后调用finishBatch；协调者bolts实现了ICommitter接口或通过TransactionalTopologyBuilder调用setCommiterBolt设置Committer bolt，与BatchBolt不同的是，finishBatch在提交就绪时（所有事务都成功提交之后）执行。
* 继承BaseTransactionalBolt并实现ICommitter接口实现Committer Bolts。
* finishBatch时需要保存最后提交的事务ID`multi.set(LAST_COMMITED_TRANSACTION_FIELD, currentTransaction);`。
* 继承BasePartitionedTransactionalSpout来实现分区的Transactional Spouts；`无论一个tuple重发多少次，它都在同一个批次里面，都有同样的事务ID；一个tuple不会出现在两个以上的批次里`。
* `IOpaquePartitionedTransactionalSpout不是等待消息中间件故障恢复，而是先读取可读的partition，因此，当某一批次消息消费失败需要重发且恰巧消息中间件故障时，会将这一批次消息发送到另一个批次，txid会改变。`

