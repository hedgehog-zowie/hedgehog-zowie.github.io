---
title: avro-introduce
date: 2018-03-21 00:00:00
toc: true
tags:
- avro
categories:
- avro
---

。

介绍并比较Avro、Protocol Buffer、Thrift三个序列化框架。

# Avro

如下内容翻译（渣翻）自[官网](http://avro.apache.org/docs/current/)。

## 介绍

Apache Avro^TM^是一个数据序列化系统，它提供如下功能：

* 丰富的数据结构。
* 紧凑、快速、二进制的数据格式化方法。
* 用于存储持久化数据的容器文件。
* 远程过程调用。
* 容易与动态语言集成。不需要生成代码，也不需要使用或实现RPC协议即可使用。生成代码作为一个可选项，仅在使用静态语言时才值得使用。

## Schemas

Avro依赖schemas，当avro数据存储为文件时，它的schema与数据一同存储，这样数据可以使用任何程序进行处理，如果程序数据希望使用一个不同的schema读取数据，也很容易解决，只需要两个schema都存在即可。

当avro用于RPC时，客户端和服务端的握手时交换schema。当客户端和服务端都拥有对方完整的schema后，很容易处理双方相同的字段、缺失的字段、额外的字段等。

##　与其他序列化框架的比较

Avro提供了Protocal Buffer、Thrift相似的功能，它们的不同主要体现在如下几个方面：

* 动态类型：Avro不需要生成代码，数据与schema绑定在一起，无需代码生成和静态语言即可完全处理数据，这使得一般的数据处理系统和语言更容易解释avro数据。
* 未标记的数据：读取数据时只需要提供schema，只有相当少的类型信息需要与数据一同编码，得到更小的序列化大小。
* 无手动指定的字段ID：当schema改变时，处理数据时，老的schema和新的schema都需要提供，因此可以使用字段名处理数据差异。

## 实例



# Protocol Buffer

# Thrift



​