---
layout:     post
title:      记一次处理ZooKeeper和Kafka的问题
subtitle:   
date:       2019-01-29
author:     BY KiloMeter
header-img: img/2019-01-29-记一次处理ZooKeeper和Kafka的问题/22.jpg
catalog: true
tags:
    - Zookeeper
    - kafka
---


主要是在做单车微信小程序+大数据分析这个项目时，需要用到nginx直接对接kafka。

问题一：配置好环境后，kafka启动不了，查看kafka日志后，报错如下

![](/img/2019-01-29-记一次处理ZooKeeper和Kafka的问题/kafka启动brokers已被注册.png)

说brokers/ids/1 已被注册，我查了下，网上的博客说法是以下这几种，

第一种是kafka的server.properties配置文件中broker.id出现了重复，需要给每台机器设置不同的brokerid。

我查了下，我的三台虚拟机上的brokerid是互不相同的。

第二种说是修改kafka的log文件夹下的meta.properties，这个文件的位置在kafka配置文件中log.dir文件夹下面，默认是在/tmp/kafka_logs/meta.properties 这个位置，最好把位置给修改下，因为tmp是临时文件目录，可能会是不是被系统给清除掉。我看了下meta.properties这个文件，里面也有一个broker.id，发现确实和自己kafka的serv.properties配置文件中的broker.id没有匹配上，我直接修改了log.dir文件夹，重启了kafka和zookeeper就好了。





问题二：在kafka中创建了新的topic后，用describe查询后发现每个分区的leader为null。

查了很久，发现问题是在以下这几个上面：

第一，确保每个节点的kafka都有成功启动，要查看是否有成功启动完成，可以在zookeeper的客户端上，使用ls /brkers/ids  显示的结果是每台机器的broker.id如果数量有误的话重启机器，确保每台机器都在正常工作。

第二，在确保每台机器都启动的情况下，先把之前创建的topic删除掉，可以在zookeeper使用 rmr /brokers/topics/topic名 进行删除，然后再重新创建。我这边在删除topic的时候有些玄学，已经选择了删除topicA，但是执行命令后查询仍然查询得到topicA，然后试着删除tipicB，查询后确实显示已经删除了的。多试几次后才能删掉。删掉topic后，再重新创建新的topic，创建完后再用describe查看，我这边这样处理后就行了。

然后可以开个生产者和消费者测试下

问题三：kafka发送消息报错，报错如下

![](/img/2019-01-29-记一次处理ZooKeeper和Kafka的问题/kafka发送消息报错.png)



问题就在于kafka集群没能成功启动，解决方法看问题二。



还有一些需要注意的小问题：

kafka的配置文件中，zookeeper.connect=localhost:2181

集群中broker.id不能重复，不能重复，不能重复，而且要和配置文件中log.dir位置下的meta.properties中的broker.id保持一致。

advertised.host.name=192.168.\*\*.\*\*  每台机器在这个位置填上自己的ip地址。

port=9092端口号