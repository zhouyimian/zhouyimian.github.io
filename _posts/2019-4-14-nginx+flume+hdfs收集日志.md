---
layout:     post
title:      nginx+flume+hdfs收集日志
subtitle:   
date:       2019-4-14
author:     BY KiloMeter
header-img: img/2019-4-14-nginx+flume+hdfs收集日志/38.jpg
catalog: true
tags:
    - 项目
---
## nginx的基本配置

nginx的下载安装这里就不介绍了，这里主要介绍下nginx的配置文件如何进行配置。

如果是使用yum安装的话，nginx的配置文件所在位置是 /usr/local/nginx/conf/nginx.conf

因为主要是为了处理nginx和flume进行对接，因此配置文件只配置了http部分

首先需要在http内部设置日志格式

![](/img/2019-4-14-nginx+flume+hdfs收集日志/nginx日志格式.png)

\$remote_addr与\$http_x_forwarded_for用以记录客户端的ip地址；

\$remote_user：用来记录客户端用户名称；

\$time_local： 用来记录访问时间与时区；

\$request： 用来记录请求的url与http协议；

\$status： 用来记录请求状态；成功是200，

\$body_bytes_sent ：记录发送给客户端文件主体内容大小；

\$http_referer：用来记录从哪个页面链接访问过来的；

还有比较重要的是设置http内部server的内容

![](/img/2019-4-14-nginx+flume+hdfs收集日志/server配置.png)

listen指定端口号

server_name 服务器名字，也可以是IP

location这个指定了要响应什么格式的请求，这里指定的是以.png结尾的请求

access_log指定了日志的保存位置，同时还要指定要使用什么日志格式(这里的lf对应第一张图片中log_format lf)

root指定了nginx根目录的位置。

其他关于nginx的详细配置可以看这篇博客

[Nginx配置文件（nginx.conf）配置详解](https://blog.csdn.net/tjcyjd/article/details/50695922)

nginx一些简单命令：

启动nginx：sbin/nginx

停止nginx：sbin/nginx -s stop

## 配置flume

首先要配置flume中sources，channels，sink

这里的配置文件具体如下

```
agent.sources=r1   
agent.sinks=k1
agent.channels=c1

agent.sources.r1.channels=c1
agent.sinks.k1.channel=c1

agent.sources.r1.type=exec
agent.sources.r1.command=tail -F /usr/local/log/access.log


agent.channels.c1.type=memory
agent.channels.c1.capacity=1000
agent.channels.c1.transactionCapacity=1000
agent.channels.c1.byteCapacityBufferPercentage=20
agent.channels.c1.byteCapacity=1000000
agent.channels.c1.keep-alive=60

agent.sinks.k1.type=hdfs
agent.sinks.k1.channel=c1
//设定hdfs存储位置
agent.sinks.k1.hdfs.path=hdfs://niubike1:9000/logs/%Y/%m/%d
//存储文件的格式(这里是纯文本的格式)
agent.sinks.k1.hdfs.fileType=DataStream
//文件名前缀
agent.sinks.k1.hdfs.filePrefix=BF-%H
//文件名后缀
agent.sinks.k1.hdfs.fileSuffix=.log
agent.sinks.k1.hdfs.minBlockReplicas=1
//设置文件的滚动时间，超出该时间会新建文件
agent.sinks.k1.hdfs.rollInterval=3600
//设置文件的滚动大小，超出大小会新建文件
agent.sinks.k1.hdfs.rollSize=132692539
agent.sinks.k1.hdfs.idleTimeout=10
agent.sinks.k1.hdfs.batchSize=1
agent.sinks.k1.hdfs.rollCount=0
agent.sinks.k1.hdfs.round=true
agent.sinks.k1.hdfs.roundValue=2
agent.sinks.k1.hdfs.roundUnit=minute
agent.sinks.k1.hdfs.useLocalTimeStamp=true
```

flume的详细配置可以参考flume 的官网文档

[**Apache Flume官方文档**™](https://flume.apache.org/FlumeUserGuide.html)

然后把hdfs有关jar包移动到flume的lib目录下

![](/img/2019-4-14-nginx+flume+hdfs收集日志/hdfs移动至flume的jar包.png)

启动flume(注意，因为这里使用了HDFS，所以要先确保hdfs已经开启)

bin/flume-ng agent --conf ./conf/ --conf-file ./conf/nginx.conf --name agent

启动后，在浏览器上访问nginx的资源，日志存储到本地后，再通过flume上传至HDFS

![](/img/2019-4-14-nginx+flume+hdfs收集日志/HDFS文件.png)(里面有一个tmp文件是因为HDFS还在写入的过程中，不小心关掉了flume导致的)。

到此，nginx，flume和hdfs已经连通，通过这个过程可以实现将日志上传至HDFS，其实flume和hdfs中间还能加上一层kafka，可以预防线上数据的突发性导致写入HDFS速度过慢导致写入数据失败的问题，这个的实现等之后再来写。