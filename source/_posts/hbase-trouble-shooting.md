---
title: hbase-trouble-shooting
date: 2016-09-27 14:42:53
toc: true
tags: 
- HBase
categories: 
- HBase
---

记录使用HBase过程中遇到的问题、原因及解决办法。

# HBase元数据修复
HBase中出现好几张表都读取报错的情况，表现为提示如下错误：

```
Mon Sep 26 16:34:49 CST 2016, org.apache.hadoop.hbase.client.RpcRetryingCaller@7293da04, org.apache.hadoop.hbase.NotServingRegionException: org.apache.hadoop.hbase.NotServingRegionException: Region BASE_USER_ACC,20160705688835318555647,1470888379980.fd89999e1f6fb506fa69fee69837d47b. is not online
        at org.apache.hadoop.hbase.regionserver.HRegionServer.getRegionByEncodedName(HRegionServer.java:2612)
        at org.apache.hadoop.hbase.regionserver.HRegionServer.getRegion(HRegionServer.java:4003)
        at org.apache.hadoop.hbase.regionserver.HRegionServer.scan(HRegionServer.java:3002)
        at org.apache.hadoop.hbase.protobuf.generated.ClientProtos$ClientService$2.callBlockingMethod(ClientProtos.java:26929)
        at org.apache.hadoop.hbase.ipc.RpcServer.call(RpcServer.java:2185)
        at org.apache.hadoop.hbase.ipc.RpcServer$Handler.run(RpcServer.java:1889)
```

重启HBase后，问题依旧。
使用命令：`hbase hbck BASE_USER_INFO_MERGE`，对该表进行一致性检查，结果如下：

```
Summary:
  hbase:meta is okay.
    Number of regions: 1
    Deployed on:  bis-hadoop-datanode-s2d-136,60020,1474869133049
Table BASE_USER_INFO_MERGE is inconsistent.
    Number of regions: 76
    Deployed on:  bis-hadoop-datanode-s-01,60020,1474869134748 bis-hadoop-datanode-s-02,60020,1474869134770 ...... bis-newdatawork-s2c-126,60020,1474869134031
9 inconsistencies detected.
Status: INCONSISTENT
```

存在不一致的情况，进一步查看一致性检查的输出日志，发现提示有一些错误如下：

```
ERROR: Region { meta => BASE_USER_INFO_MERGE,86188803131181,1472665253414.0c9057eab1d7da58f2d44bfab641dada., hdfs => hdfs://webcluster/hbase/data/default/BASE_USER_INF
O_MERGE/0c9057eab1d7da58f2d44bfab641dada, deployed =>  } not deployed on any region server.
ERROR: Region { meta => BASE_USER_INFO_MERGE,860920020927537,1472665222872.1acfd28e4be4fb26da89c3db11d83436., hdfs => hdfs://webcluster/hbase/data/default/BASE_USER_IN
FO_MERGE/1acfd28e4be4fb26da89c3db11d83436, deployed =>  } not deployed on any region server.
ERROR: Region { meta => BASE_USER_INFO_MERGE,94de2654afe507bd,1473448496650.2807f8fa14d34f7d997b571c00000366., hdfs => hdfs://webcluster/hbase/data/default/BASE_USER_I
NFO_MERGE/2807f8fa14d34f7d997b571c00000366, deployed =>  } not deployed on any region server.
ERROR: Region { meta => BASE_USER_INFO_MERGE,862029023490010,1472665253414.39df136d1fb6eda60bbd06d0150f4f01., hdfs => hdfs://webcluster/hbase/data/default/BASE_USER_IN
FO_MERGE/39df136d1fb6eda60bbd06d0150f4f01, deployed =>  } not deployed on any region server.
ERROR: Region { meta => BASE_USER_INFO_MERGE,fa7df192ea2395ad,1472871226021.59bc8c67cc00216964a36188d10223e0., hdfs => hdfs://webcluster/hbase/data/default/BASE_USER_I
NFO_MERGE/59bc8c67cc00216964a36188d10223e0, deployed =>  } not deployed on any region server.
2016-09-27 14:31:43,978 DEBUG [main] util.HBaseFsck: There are 82 region info entries
2016-09-27 14:31:44,685 INFO  [main] util.HBaseFsck: Handling overlap merges in parallel. set hbasefsck.overlap.merge.parallel to false to run serially.
ERROR: There is a hole in the region chain between 860920020927537 and 861060011050756.  You need to create a new .regioninfo and region dir in hdfs to plug the hole.
ERROR: There is a hole in the region chain between 86188803131181 and 862222012466956.  You need to create a new .regioninfo and region dir in hdfs to plug the hole.
ERROR: There is a hole in the region chain between 94de2654afe507bd and 9f2bdf5.  You need to create a new .regioninfo and region dir in hdfs to plug the hole.
ERROR: Last region should end with an empty key. You need to create a new region and regioninfo in HDFS to plug the hole.
```

错误提示为：该表存在一些region未分配到任何regionserver上，且该表存在空洞，即region不连续的情况。
于是，进一步执行命令修复空洞：`hbase hbck -repairHoles BASE_USER_INFO_MERGE`，可以看到类似如下输出：

```
ERROR: Region { meta => BASE_USER_INFO_MERGE,862029023490010,1472665253414.39df136d1fb6eda60bbd06d0150f4f01., hdfs => hdfs://webcluster/hbase/data/default/BASE_USER_IN
FO_MERGE/39df136d1fb6eda60bbd06d0150f4f01, deployed =>  } not deployed on any region server.
Trying to fix unassigned region...
```

然而，最后的修复结果：

```
Summary:
  hbase:meta is okay.
    Number of regions: 1
    Deployed on:  bis-hadoop-datanode-s2d-136,60020,1474869133049
Table BASE_USER_INFO_MERGE is inconsistent.
    Number of regions: 76
    Deployed on:  ......
9 inconsistencies detected.
```

问题依旧，尝试在HBase中删除一个分区的元数据记录：`deleteall 'hbase:meta', 'BASE_USER_INFO_MERGE,86188803131181,1472665253414.0c9057eab1d7da58f2d44bfab641dada.'`

然后，再重新生成：`hbase hbck -fixMeta BASE_USER_INFO_MERGE `，此时，元数据重新生成后，该region处于`not deployed`状态，再执行命令将该region重新分配：`hbase hbck -fixAssignments BASE_USER_INFO_MERGE`，然而，却无法重新分配该region，无奈，只得删除其元数据及HDFS上的相关文件，之后再重新生成元数据，并修补空洞。
如下：
```
deleteall 'hbase:meta', 'BASE_USER_INFO_MERGE,860920020927537,1472665222872.1acfd28e4be4fb26da89c3db11d83436.'
```
```
hdfs dfs -rm -r -skipTrash /hbase/data/default/BASE_USER_INFO_MERGE/1acfd28e4be4fb26da89c3db11d83436
```
```
hbase hbck -repairHoles BASE_USER_INFO_MERGE
```
最后再检查一致性`hbase hbck BASE_USER_INFO_MERGE`：
```
Summary:
  hbase:meta is okay.
    Number of regions: 1
    Deployed on:  bis-hadoop-datanode-s2d-136,60020,1474869133049
  BASE_USER_INFO_MERGE is okay.
    Number of regions: 81
    Deployed on:  ......
0 inconsistencies detected.
Status: OK
```

# hbase regionserver异常退出
部分hbase regionserver异常退出，摘取部分日志如下：
```
2016-10-11 18:57:36,917 WARN  [regionserver60020] util.Sleeper: We slept 38237ms instead of 3000ms, this is likely due to a long garbage collecting pause and it's usually bad, see http://hbase.apache.org/book.html#trouble.rs.runtime.zkexpired
2016-10-11 18:57:36,914 WARN  [JvmPauseMonitor] util.JvmPauseMonitor: Detected pause in JVM or host machine (eg GC): pause of approximately 37257ms
GC pool 'ParNew' had collection(s): count=1 time=37664ms
...
2016-10-11 18:57:36,913 WARN  [ResponseProcessor for block BP-45131683-16.6.10.141-1430914429099:blk_1286873659_213203810] hdfs.DFSClient: DFSOutputStream ResponseProcessor exce
ption  for block BP-45131683-16.6.10.141-1430914429099:blk_1286873659_213203810
java.io.EOFException: Premature EOF: no length prefix available
        at org.apache.hadoop.hdfs.protocolPB.PBHelper.vintPrefixed(PBHelper.java:1492)
        at org.apache.hadoop.hdfs.protocol.datatransfer.PipelineAck.readFields(PipelineAck.java:116)
        at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer$ResponseProcessor.run(DFSOutputStream.java:721)
2016-10-11 18:57:36,913 WARN  [regionserver60020.periodicFlusher] util.Sleeper: We slept 39267ms instead of 10000ms, this is likely due to a long garbage collecting pause and it
's usually bad, see http://hbase.apache.org/book.html#trouble.rs.runtime.zkexpired
2016-10-11 18:57:36,925 WARN  [DataStreamer for file /hbase/WALs/bis-newdatanode-s2b-72,60020,1474425646831/bis-newdatanode-s2b-72%2C60020%2C1474425646831.1476182840215 block BP
-45131683-16.6.10.141-1430914429099:blk_1286873659_213203810] hdfs.DFSClient: Error Recovery for block BP-45131683-16.6.10.141-1430914429099:blk_1286873659_213203810 in pipeline
 10.10.10.103:50010, 10.10.10.50:50010, 10.10.10.147:50010: bad datanode 10.10.10.103:50010
2016-10-11 18:57:36,934 FATAL [regionserver60020] regionserver.HRegionServer: ABORTING region server bis-newdatanode-s2b-72,60020,1474425646831: org.apache.hadoop.hbase.YouAreDe
adException: Server REPORT rejected; currently processing bis-newdatanode-s2b-72,60020,1474425646831 as dead server
```
同时在gc.log中可以看到：
```
2016-10-12T15:25:43.201+0800: 1774.137: [GC 1622131K->787661K(4089472K), 16.5654980 secs]
2016-10-12T15:26:49.921+0800: 1840.857: [GC 1626573K->809761K(4089472K), 0.2250050 secs]
2016-10-12T15:27:58.027+0800: 1908.963: [GC 1648673K->818686K(4089472K), 35.9152720 secs]
```
从第一部分日志和gc.log，可以看出是垃圾回收耗时太长引起regionserver与zookeeper连接超时，从而被认作是一个死节点，并中断（ABORTING）该regionserver。

## 解决办法
首先根据日志提示，查看http://hbase.apache.org/book.html#trouble.rs.runtime.zkexpired，其中的建议如下：
* Make sure you give plenty of RAM (in hbase-env.sh), the default of 1GB won’t be able to sustain long running imports.
* Make sure you don’t swap, the JVM never behaves well under swapping.
* Make sure you are not CPU starving the RegionServer thread. For example, if you are running a MapReduce job using 6 CPU-intensive tasks on a machine with 4 cores, you are probably starving the RegionServer enough to create longer garbage collection pauses.
* Increase the ZooKeeper session timeout

简单翻译一下：
* 确保有足够的内存
* 禁用swap
* 确保有足够的CPU可供regionserver使用
* 增大zookeeper会话超时时间

其中关于增大zookeeper会话超时时间，官网给出了如下配置：
```
If you wish to increase the session timeout, add the following to your hbase-site.xml to increase the timeout from the default of 60 seconds to 120 seconds.

<property>
  <name>zookeeper.session.timeout</name>
  <value>1200000</value><!--这里应该是多写了一个0-->
</property>
<property>
  <name>hbase.zookeeper.property.tickTime</name>
  <value>6000</value>
</property>
```

然而在hbase-site.xml中配置了该参数重启集群后，并未生效，查看日志：
```
2016-10-13 15:55:22,599 INFO  [SplitLogWorker-bis-newdatanode-s2b-60,60020,1476345314703-SendThread(bis-newdatanode-s2b-58:2181)] zookeeper.ClientCnxn: Opening socket connection
 to server bis-newdatanode-s2b-58/10.10.10.73:2181. Will not attempt to authenticate using SASL (unknown error)
2016-10-13 15:55:22,603 INFO  [SplitLogWorker-bis-newdatanode-s2b-60,60020,1476345314703-SendThread(bis-newdatanode-s2b-58:2181)] zookeeper.ClientCnxn: Socket connection establi
shed to bis-newdatanode-s2b-58/10.10.10.73:2181, initiating session
2016-10-13 15:55:22,605 INFO  [SplitLogWorker-bis-newdatanode-s2b-60,60020,1476345314703-SendThread(bis-newdatanode-s2b-58:2181)] zookeeper.ClientCnxn: Session establishment com
plete on server bis-newdatanode-s2b-58/10.10.10.73:2181, sessionid = 0x757bc4782260012, negotiated timeout = 40000
```
显示其最大超时时间仍旧为40000ms，究其原因，是zookeeper服务端的启动时，会设置最大超时时间，其算法如下：
```
public int getMinSessionTimeout(){ 
  return minSessionTimeout == -1 ? tickTime * 2 : minSessionTimeout; 
} 
public int getMaxSessionTimeout() {
  return maxSessionTimeout == -1 ? tickTime * 20 : maxSessionTimeout; 
}
```
默认情况，tickTime=2sec，那么minSessionTimeout 和 maxSessionTimeout 分别是4sec和40sec。

在zoo.cfg中添加配置：
```
maxSessionTimeout = 300000
```
重启，查看zookeeper输出：
```
2016-10-14 09:30:50,963 [myid:1] - INFO  [main:QuorumPeer@959] - tickTime set to 2000
2016-10-14 09:30:50,964 [myid:1] - INFO  [main:QuorumPeer@979] - minSessionTimeout set to -1
2016-10-14 09:30:50,964 [myid:1] - INFO  [main:QuorumPeer@990] - maxSessionTimeout set to 300000
```

至此，问题解决。

## 野蛮的解决办法

所谓野蛮，即不去深究zk租约超期的具体原因，而是在遇到这种情况而导致regionserver中断时，直接重新启动regionserver。
具体措施为在配置文件`hbase-site.xml`中添加配置如下：
```
<property>
<name>hbase.regionserver.restart.on.zk.expire</name>
<value>true</value>
<description>
Zookeeper session expired will force regionserver exit.
Enable this will make the regionserver restart.
</description>
```

