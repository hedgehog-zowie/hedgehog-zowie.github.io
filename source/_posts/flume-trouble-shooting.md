---
title: flume-trouble-shooting
date: 2016-10-14 18:26:51
toc: true
tags: 
- Flume
categories: 
- Flume
---

记录使用Flume过程中遇到的问题、原因及解决办法。

# java.io.IOException: Bad response ERROR for block
flume处理日志时，偶尔报出如下错误:
```
gdata 2016-10-13 10:33:01,133 (ResponseProcessor for block BP-45131683-16.6.10.141-1430914429099:blk_1288600226_214984186) [WARN - org.apache.hadoop.hdfs.DFSOutputStream$DataStr
eamer$ResponseProcessor.run(DFSOutputStream.java:774)] DFSOutputStream ResponseProcessor exception  for block BP-45131683-16.6.10.141-1430914429099:blk_1288600226_214984186
java.io.IOException: Bad response ERROR for block BP-45131683-16.6.10.141-1430914429099:blk_1288600226_214984186 from datanode 10.10.10.71:50010
        at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer$ResponseProcessor.run(DFSOutputStream.java:732)
gdata 2016-10-13 10:33:01,137 (DataStreamer for file /data/flume/event_session/day=20161013/uptime=2016101307/event-session.1476325736178.tmp block BP-45131683-16.6.10.141-14309
14429099:blk_1288600226_214984186) [WARN - org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer.setupPipelineForAppendOrRecovery(DFSOutputStream.java:1013)] Error Recovery for bl
ock BP-45131683-16.6.10.141-1430914429099:blk_1288600226_214984186 in pipeline 10.10.10.103:50010, 10.10.10.71:50010, 10.10.10.35:50010: bad datanode 10.10.10.71:50010
```
查看对应datanode —— 10.10.10.71上的日志：
```
2016-10-13 10:33:01,130 INFO org.apache.hadoop.hdfs.server.datanode.DataNode: Exception for BP-45131683-16.6.10.141-1430914429099:blk_1288600226_214984186
java.net.SocketTimeoutException: 60000 millis timeout while waiting for channel to be ready for read. ch : java.nio.channels.SocketChannel[connected local=/10.10.10.71:50010 rem
ote=/10.10.10.103:46888]
        at org.apache.hadoop.net.SocketIOWithTimeout.doIO(SocketIOWithTimeout.java:164)
        at org.apache.hadoop.net.SocketInputStream.read(SocketInputStream.java:161)
        at org.apache.hadoop.net.SocketInputStream.read(SocketInputStream.java:131)
        at java.io.BufferedInputStream.fill(BufferedInputStream.java:235)
        at java.io.BufferedInputStream.read1(BufferedInputStream.java:275)
        at java.io.BufferedInputStream.read(BufferedInputStream.java:334)
        at java.io.DataInputStream.read(DataInputStream.java:149)
        at org.apache.hadoop.io.IOUtils.readFully(IOUtils.java:192)
        at org.apache.hadoop.hdfs.protocol.datatransfer.PacketReceiver.doReadFully(PacketReceiver.java:213)
        at org.apache.hadoop.hdfs.protocol.datatransfer.PacketReceiver.doRead(PacketReceiver.java:134)
        at org.apache.hadoop.hdfs.protocol.datatransfer.PacketReceiver.receiveNextPacket(PacketReceiver.java:109)
        at org.apache.hadoop.hdfs.server.datanode.BlockReceiver.receivePacket(BlockReceiver.java:435)
        at org.apache.hadoop.hdfs.server.datanode.BlockReceiver.receiveBlock(BlockReceiver.java:693)
        at org.apache.hadoop.hdfs.server.datanode.DataXceiver.writeBlock(DataXceiver.java:569)
        at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.opWriteBlock(Receiver.java:115)
        at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.processOp(Receiver.java:68)
        at org.apache.hadoop.hdfs.server.datanode.DataXceiver.run(DataXceiver.java:221)
        at java.lang.Thread.run(Thread.java:722)
```
显示`60000 millis timeout while waiting for channel to be ready for read`，该问题是由于datanode rpc连接都被占用，导致客户端请求超时。
因此在hdfs-site.xml中加大datanode的rpc连接数：
```
        <property>
                 <name>dfs.datanode.handler.count</name>
                 <value>10</value>
        </property>
```

# Callable timed out after 10000 ms
忘了截取日志，因此摘抄自如下网页：
http://blog.csdn.net/wsscy2004/article/details/22179361

## 解决办法：
修改flume配置文件，添加配置hdfs.callTimeout = 60000(该值默认为10000)

