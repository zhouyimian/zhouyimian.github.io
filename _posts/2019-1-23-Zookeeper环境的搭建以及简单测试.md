---
layout:     post
title:      Zookeeper环境的搭建以及简单测试
subtitle:   
date:       2019-1-23
author:     BY KiloMeter
header-img: img/2019-1-23-Zookeeper环境的搭建以及简单测试/6.jpg
catalog: true
tags:
    - Zookeeper
---


### Zookeeper环境搭建

首先去下载ZK的安装包，解压后可以看到以下目录结构

![](/img/2019-1-23-Zookeeper环境的搭建以及简单测试/Zookeeper解压目录.png)

**注：**data你们是没有的，得后面自己mkdir生成，用于存放ZK集群的各个服务器IP及端口，以及本机的编号(后面修改zoo.cfg配置文件的时候会讲)。

我这边的ZK集群使用了三台虚拟机。

要配置集群的话，进入conf文件夹，可以看到zoo_sample.cfg文件，该文件是ZK配置文件的模板，把该文件拷贝一份，命名为zoo.cfg。

修改zoo.cfg文件

![](/img/2019-1-23-Zookeeper环境的搭建以及简单测试/ZK配置文件内容.png)



dataDir默认是在/tmp文件夹下，作用如上所述，因此需要自己新建文件夹，放在/tmp下的文件会被定期清除掉。

clientPort是client的连接端口。

其他配置属性可以自行百度，对于初期的ZK集群的搭建没什么影响

**注意：**需要在配置文件下面添加集群中各个节点的信息

![](/img/2019-1-23-Zookeeper环境的搭建以及简单测试/ZK集群配置添加各节点信息.png)

server.x代表了整个服务的机器，当服务启动后，会去上面说到的的dataDir下面找一个myid文件(这个文件也需要我们在每台机器上自己创建，文件内容只需要写明本机的编号即可)，从而判断自己本机属于哪个server。

上面的2888端口号代表了集群中各个服务器与leader服务器交换信息的端口，3888代表如果leader服务器挂了之后，重新选择leader时信息交换的端口。

修改日志文件的位置在zookeeper的bin目录下的zkEnv.sh，在56行处修改

![](/img/2019-1-23-Zookeeper环境的搭建以及简单测试/ZK目录路径修改.png)

配置完成后把上面的配置信息在集群中所有的机器上进行同样的修改，但是myid需要单独设置。

```shell
bin/zkServer.sh start    #启动ZK集群

bin/zkServer.sh status   #查看集群状态
```

如果成功启动的话在三台机器上分别使用status可以看到如下(folloer和leader是通过选举选出来的，这里我们的可能会不一样，关于ZK的选举可以查看[这篇文章](https://zhouyimian.github.io/2019/01/22/Zookeeper%E6%A6%82%E5%BF%B5-%E5%8E%9F%E7%90%86-%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/))

![](/img/2019-1-23-Zookeeper环境的搭建以及简单测试/ZK集群成功启动标志.png)



**但是，查看状态有更大的可能性是看到以下情况**

![](/img/2019-1-23-Zookeeper环境的搭建以及简单测试/ZK集群启动失败.png)

如果出现这种情况，需要去查看以下ZK的启动日志文件。

查看日志后一般是以下这个错误，

```java
Cannot open channel to * at election address /192.168.43.52:3888
```

我总结了网上一般这种情况有以下几种可能性

**1、防火墙没有关**

**2、你的ZK配置文件中出现错误，可能是dataDir拼错这些低级错误(一般不会)。**

**3、dataDir目录没有创建**

**4、没有在dataDir目录下创建myid文件和myid没有和配置文件的id匹配**

**5、StackOverflow上的一种解决方法，每台机器上对应自己机器的server.*的IP地址改成0.0.0.0**

**6、启动的时候是在先去其他机器上启动机器上的Zookeeper，然后再启动myid为1的机器上的Zookeeper(前面启动的时候我是先在myid为1的机器上启动ZooKeeper的，后来使用该方法问题就解决了)**



### Zookeeper测试

**以下内容来自[Zookeeper分布式过程协同技术详解](https://book.douban.com/subject/26766807/)**

首先根据status查看到leader的机器所在，使用zkCli连接Zookeeper集群

```shell
create -e /master "niubike3:2888"   
# 创建master节点，-e代表此节点为临时节点，临时节点会在会话过期或者关闭时自动被删除 "niubike3:2888" 为主机信息，这里并不是必须的

stat /master true
# 在另一来follower机器上，同样使用zkCli连接集群，使用stat /master true来监视master，因为上面创建的master节点是临时的，一旦检测到这个master节点挂掉，需要有节点来接替主节点。通过在路径后面设置参数true添加监视点

delete /master
# 设置监视点后可以在leader上删除master，然后follower将会接收到如下信息
WatchedEvent state:SyncConnected type:NodeDeleted path:/master
```

从节点的任务、分配

先创建三个重要的父znode：/workers、/tasks、/assign

在真正的应用程序中，这三个节点可能是在主进程分配任务前创建，也有可能是由一个引导程序来创建，一旦这些节点存在了，主节点就需要对/workers和/tasks进行监视

```shell
create /workers ""
create /tasks ""
create /assign ""
ls /workers true
ls /tasks true
```



之后，每个从节点必须告知主节点自己是可以执行任务的，从节点通过在/workers节点下创建临时节点来通知主节点，并在子节点使用主机名来标识自己

```shell
create -e /workers/niubike3 "niubike3:2888"
```

接着，从节点需要在assigns节点下创建/assign/niubike3来接收任务分配，并要对该子节点进行监视

```shell
create /assign/niubike3 ""
ls /assign/niubike3 true
```

设置成功后，现在从节点是可以接收任务的，然后可以在第三台机器上模拟客户端角色进行任务添加

```shell
create -s /tasks/task- "cmd"
#添加任务后，客户端需要对该任务进行监视，因为任务一旦完成，会创建一个znode节点来表示任务状态，客户端通过查看任务状态的znode节点来确定任务是否完成
ls /tasks/task-0000000000 true
# 任务创建成功后，因为前面master主节点已经监视了tasks节点，因此会接收到消息
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/tasks
```

之后，主节点通过查看/workers节点内容，获取到可用节点列表，然后对任务进行分配

```shell
create /assign/niubike3/task-0000000000 ""
#从节点接收到消息
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/assign/niubike3
```

从节点接收到任务后，任务执行完毕之后再 /tasks/task-0000000000下面创建一个status节点

```shell
create /tasks/task-0000000000/status "done"
```

客户端接收到消息后，查看下status从而可以判断任务是否完成

```shell
get /tasks/task-0000000000/status
```

