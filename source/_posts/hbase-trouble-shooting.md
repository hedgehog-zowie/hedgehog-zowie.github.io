---
title: hbase-trouble-shooting
date: 2016-09-27 14:42:53
toc: true
tags: 
- HBase
categories: 
- HBase
---

此处记录使用HBase过程中遇到的问题、原因及解决办法。

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
