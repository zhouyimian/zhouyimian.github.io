---
layout:     post
title:      nginx+flume+hdfs+MR+HBase日志分析
subtitle:   
date:       2019-4-20
author:     BY KiloMeter
header-img: img/2019-4-20-nginx+flume+hdfs+MR+HBase日志分析/40.png
catalog: true
tags:
    - 项目
---
[nginx+flume+hdfs收集日志](https://zhouyimian.github.io/2019/04/14/nginx+flume+hdfs%E6%94%B6%E9%9B%86%E6%97%A5%E5%BF%97/)这一篇接着，开始对数据进行处理，通过Mapreduce对数据进行清洗和转换，然后把清洗后的结果写入到HBase

首先数据的获取方面，这边自己写了一个jssdk，用于产生日志数据并发送至nginx，nginx通过前面那篇博客的方法发往hdfs，这里主要讲运行时的一些错误处理。

由于这里不需要什么聚合操作，所以这里没有reduce程序，只有map程序

这里的日志数据格式为   客户端IP\^A时间戳\^A访问url\^A参数

先对日志进行分割，判断日志格式是否有效，有效的数据才保存到HBase中，判断是否有效根据业务需求来，比如必须要有时间戳之类的条件。

下面主要讲程序运行过程中出现的错误

错误1：

```java
Caused by: java.lang.VerifyError: class org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$AppendRequestProto overrides final method getUnknownFields.()Lcom/google/protobuf/UnknownFieldSet;
```

该错误是因为hbase和hadoop都依赖了protobuf这个包，发生了冲突，只需要把其中一个给移除了即可

```xml
<dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-client</artifactId>
      <version>2.5.0</version>
    </dependency>

    <dependency>
      <groupId>org.apache.hbase</groupId>
      <artifactId>hbase-client</artifactId>
      <version>0.98.6-hadoop2</version>
      <exclusions>
        <exclusion>
          <artifactId>protobuf-java</artifactId>
          <groupId>com.google.protobuf</groupId>
        </exclusion>
      </exclusions>
    </dependency>

    <dependency>
      <groupId>org.apache.hbase</groupId>
      <artifactId>hbase-server</artifactId>
      <version>0.98.6-hadoop2</version>
      <exclusions>
        <exclusion>
          <artifactId>protobuf-java</artifactId>
          <groupId>com.google.protobuf</groupId>
        </exclusion>
      </exclusions>
    </dependency>
```

错误2

![](/img/2019-4-20-nginx+flume+hdfs+MR+HBase日志分析/连接不上HBase和Zookeeper.png)

```java
java.net.ConnectException: Connection refused: no further information
java.io.IOException: Failed on local exception: java.io.IOException: Couldn't set up IO streams;
```

程序运行时一直显示连接失败，错误还提示说是连接hbase的60020端口和zookeeper失败

我这边出现该错误的主要原因是程序的hbase jar包版本太低了的，后面切换成上面这个0.98.6-hadoop2版本就解决问题了

错误3

程序运行时一直提示说找不到文件hdfs文件目录(/logs/2019/04/20/)，但是尝试了下在其他程序上运行是能够找到该路径的，一开始我以为是配置文件的位置问题，先是修改了配置文件的存放位置，放到了resource目录下，发现还是找不到路径，然后发现了配置信息的configuration对象中没有设置hdfs的路径，加上下面这句就ok了。

```java
configuration.set("fs.defaultFS","hdfs://192.168.43.50:9000");
```

错误4

程序显示有经过筛选的数据，但是在hbase中查看不到数据，且运行过程一直报错。后面发现是没有在HBase上建表导致的，建了表之后插入数据就不再报错了，也能在HBase中查询到数据。