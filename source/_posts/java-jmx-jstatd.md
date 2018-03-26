---
title: java-jmx-jstatd
date: 2018-03-21 00:00:00
toc: true
tags:
- avro
categories:
- avro
---

Java VisualVm有两种远程监控模式：JMX和jstatd。其中JMX是某一个java实例的监控，可以查看线程，对Cpu、内存进行抽样、强制进行垃圾回收；jstatd是整个java环境的监控，能自动检测被监控服务器上所有java应用程序，可以查看GC状态。下面详细介绍JMX和jstatd的设置。

# JMX

`JMX可以使用监视、线程、抽样器、MBeans、Buffer Pools、Tracer等工具，但无法使用Visual GC。`

启动程序时加上JVM参数如下：

```shell
-Dcom.sun.management.jmxremote=true
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
```

`注意：需要使用hostname -i检查机器IP，也可使用参数java.rmi.server.hostname指定IP，如：`

```shell
-Djava.rmi.server.hostname=192.168.0.1
```

若要使用权限配置，需要配置jmxremote.password 及 jmxremote.access `（这两个文件通常位于${JAVA_HOME}/jre/lib/management/目录下）`，一个用于记录账号密码，一个用于设置权限，然后将上述JVM参数中的`com.sun.management.jmxremote.authenticate`参数置为`true`。

# jstatd

`jstatd可以使用监视（但无法查看CPU使用状态）、Visual GC、Tracer等工具。`

在远程主机上，创建jstatd.all.policy文件，内容为：

```shell
grant codebase "file:${java.home}/../lib/tools.jar" {  
     permission java.security.AllPermission;
};
```

然后启动jstatd：

```shell
jstatd -p 40123 -J-Djava.security.policy=jstatd.all.policy
```

`注意：需要使用hostname -i检查机器IP，也可使用参数java.rmi.server.hostname指定IP，如：`

```shell
jstatd -p 40123 -J-Djava.rmi.server.hostname=192.168.0.2 -J-Djava.security.policy=jstatd.all.policy
```

