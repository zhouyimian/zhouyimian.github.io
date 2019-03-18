---
layout:     post
title:      Zookeeper概念、原理、应用场景
subtitle:   
date:       2019-1-22
author:     BY KiloMeter
header-img: img/2019-1-22-Zookeeper概念、原理、应用场景/9.jpg
catalog: true
tags:
    - Zookeeper
---

文章前面部分转自segmentfault    [**原文链接**](https://segmentfault.com/a/1190000014479433)

最后Zookeeper使用场景转自掘金  [**原文链接**](https://juejin.im/post/5a35d45251882538d31001d2)

还有这篇文章对于ZooKeeper的概念，原理都解释的很不错(个人感觉，如果再贴过来的话本文就有点长了 所有这里直接放链接)   [文章链接](https://www.cnblogs.com/sunddenly/p/4138580.html)

## 1、什么是Zookeeper

Zookeeper是一个**分布式**的，开放源码的分布式**应用程序协调服务**，是Google的Chubby一个开源的实现，它是**集群的管理者**，**监视着集群中各个节点的状态根据节点提交的反馈进行下一步合理操作**。最终，将简单易用的接口和性能高效、功能稳定的系统提供给用户。
客户端的**读请求**可以被集群中的**任意一台机器处理**，如果读请求在节点上注册了监听器，这个监听器也是由所连接的Zookeeper机器来处理。对于**写请求**，这些请求会同**时发给其他Zookeeper机器并且达成一致后，请求才会返回成功**。因此，随着**Zookeeper的集群机器增多，读请求的吞吐会提高但是写请求的吞吐会下降**。
有序性是Zookeeper中非常重要的一个特性，所有的**更新都是全局有序的**，每个更新都有一个**唯一的时间戳**，这个时间戳称为**zxid（Zookeeper Transaction Id）**。而**读请求只会相对于更新有序**，也就是读请求的返回结果中会带有这个**Zookeeper最新的zxid**。

## 2、Zookeeper提供了什么

1、**文件系统**

2、**通知系统**

## 3、Zookeeper文件系统

* Zookeeper提供一个多层级的节点命名空间（节点称为znode）。与文件系统不同的是，这些节点**都可以设置关联的数据**，而文件系统中只有文件节点可以存放数据而目录节点不行。Zookeeper为了保证高吞吐和低延迟，在内存中维护了这个树状的目录结构，这种特性使得Zookeeper**不能用于存放大量的数据**，每个节点的存放数据上限为**1M**。
* 全局数据一致：每个server保存一份相同的数据副本，client无论连接到哪个server，数据都是一致的

## 4、四种类型的znode

1、**PERSISTENT-持久化目录节点** 
客户端与Zookeeper断开连接后，该节点依旧存在 
2、**PERSISTENT_SEQUENTIAL-持久化顺序编号目录节点**
客户端与Zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号 
3、**EPHEMERAL-临时目录节点**
客户端与Zookeeper断开连接后，该节点被删除 
4、**EPHEMERAL_SEQUENTIAL-临时顺序编号目录节点**
客户端与Zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编号



## 5、Zookeeper通知机制

client端会对某个znode建立一个**watcher事件**，当该znode发生变化时，这些client会收到Zookeeper的通知，然后client可以根据znode变化来做出业务上的改变等。

## 6、Zookeeper做了什么？

1、命名服务
2、配置管理
3、集群管理
4、分布式锁
5、队列管理

## 7、Zookeeper的命名服务（文件系统）

命名服务是指通过指定的名字来**获取资源**或者**服务的地址**，利用Zookeeper创建一个全局的路径，即是**唯一**的路径，这个路径就可以作为一个名字，指向集群中的集群，提供的服务的地址，或者一个远程的对象等等。

## 8、Zookeeper的配置管理（文件系统、通知机制）

程序分布式的部署在不同的机器上，将程序的配置信息放在Zookeeper的**znode**下，当有配置发生改变时，也就是znode发生变化时，可以通过改变Zookeeper中某个目录节点的内容，利用**watcher**通知给各个客户端，从而更改配置。

## 9、Zookeeper集群管理（文件系统、通知机制）

所谓集群管理无在乎两点：**是否有机器退出和加入、选举master**。 
对于第一点，所有机器约定在父目录下**创建临时目录节点**，然后监听父目录节点的子节点变化消息。一旦有机器挂掉，该机器与 Zookeeper的连接断开，其所创建的临时目录节点被删除，**所有其他机器都收到通知：某个兄弟目录被删除**，于是，所有人都知道：它上船了。
新机器加入也是类似，**所有机器收到通知：新兄弟目录加入**，highcount又有了，对于第二点，我们稍微改变一下，**所有机器创建临时顺序编号目录节点，每次选取编号最小的机器作为master就好**。

## 10、Zookeeper分布式锁（文件系统、通知机制）

有了Zookeeper的一致性文件系统，锁的问题变得容易。锁服务可以分为两类，一个是**保持独占**，另一个是**控制时序**。 
对于第一类，我们将Zookeeper上的一个**znode看作是一把锁**，通过createznode的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。用完删除掉自己创建的distribute_lock 节点就释放出锁。 
对于第二类， /distribute_lock 已经预先存在，所有客户端在它下面创建临时顺序编号目录节点，和选master一样，**编号最小的获得锁**，用完删除，依次方便。

## 11、获取分布式锁的流程



![](/img/2019-1-22-Zookeeper概念、原理、应用场景/获取分布式锁的流程1.png)



在获取分布式锁的时候在locker节点下创建临时顺序节点，释放锁的时候删除该临时节点。客户端调用createNode方法在locker下创建临时顺序节点，
然后调用getChildren(“locker”)来获取locker下面的所有子节点，注意此时不用设置任何Watcher。客户端获取到所有的子节点path之后，如果发现自己创建的节点在所有创建的子节点序号最小，那么就认为该客户端获取到了锁。如果发现自己创建的节点并非locker所有子节点中最小的，说明自己还没有获取到锁，此时客户端需要找到**比自己小的那个节点**，然后对其调用**exist()**方法，同时对其注册事件监听器。之后，让这个被关注的节点删除，则客户端的Watcher会收到相应通知，此时再次判断自己创建的节点是否是locker子节点中序号最小的，如果是则获取到了锁，如果不是则重复以上步骤继续获取到比自己小的一个节点并注册监听。当前这个过程中还需要许多的逻辑判断。

![](/img/2019-1-22-Zookeeper概念、原理、应用场景/获取分布式锁的流程2.png)



代码的实现主要是基于互斥锁，获取分布式锁的重点逻辑在于**BaseDistributedLock**，实现了基于Zookeeper实现分布式锁的细节。

## 12、Zookeeper队列管理（文件系统、通知机制）

两种类型的队列：
1、同步队列，当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达。 
2、队列按照 FIFO 方式进行入队和出队操作。 
第一类，在约定目录下创建临时目录节点，监听节点数目是否是我们要求的数目。 
第二类，和分布式锁服务中的控制时序场景基本原理一致，入列有编号，出列按编号。在特定的目录下创建**PERSISTENT_SEQUENTIAL**节点，创建成功时**Watcher**通知等待的队列，队列删除**序列号最小的节点**用以消费。此场景下Zookeeper的znode用于消息存储，znode存储的数据就是消息队列中的消息内容，SEQUENTIAL序列号就是消息的编号，按序取出即可。由于创建的节点是持久化的，所以**不必担心队列消息的丢失问题**。

## 13、Zookeeper数据复制

Zookeeper作为一个集群提供一致的数据服务，自然，它要在**所有机器间**做数据复制。数据复制的好处： 
1、容错：一个节点出错，不致于让整个系统停止工作，别的节点可以接管它的工作； 
2、提高系统的扩展能力 ：把负载分布到多个节点上，或者增加节点来提高系统的负载能力； 
3、提高性能：让**客户端本地访问就近的节点，提高用户访问速度**。

从客户端读写访问的透明度来看，数据复制集群系统分下面两种： 
1、**写主**(WriteMaster) ：对数据的**修改提交给指定的节点**。读无此限制，可以读取任何一个节点。这种情况下客户端需要对读与写进行区别，俗称**读写分离**； 
2、**写任意**(Write Any)：对数据的**修改可提交给任意的节点**，跟读一样。这种情况下，客户端对集群节点的角色与变化透明。

对Zookeeper来说，它采用的方式是**写任意**。通过增加机器，它的读吞吐能力和响应能力扩展性非常好，而写，随着机器的增多吞吐能力肯定下降（这也是它建立observer的原因），而响应能力则取决于具体实现方式，是**延迟复制保持最终一致性**，还是**立即复制快速响应**。

## 14、Zookeeper工作原理

Zookeeper 的核心是**原子广播**，这个机制保证了**各个Server之间的同步**。实现这个机制的协议叫做**Zab协议**。Zab协议有两种模式，它们分别是**恢复模式（选主）**和**广播模式（同步）**。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和 leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

## 15、Zookeeper是如何保证事务的顺序一致性的？

Zookeeper采用了**递增的事务Id**来标识，所有的proposal（提议）都在被提出的时候加上了zxid，zxid实际上是一个64位的数字，高32位是epoch（时期; 纪元; 世; 新时代）用来标识leader是否发生改变，如果有新的leader产生出来，epoch会自增，**低32位用来递增计数**。当新产生proposal的时候，会依据数据库的两阶段过程，首先会向其他的server发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行。

## 16、Zookeeper 下 Server工作状态

每个Server在工作过程中有三种状态： 
LOOKING：当前Server**不知道leader是谁**，正在搜寻
LEADING：当前Server即为选举出来的leader
FOLLOWING：leader已经选举出来，当前Server与之同步

## 17、Zookeeper是如何选取主leader的？

当leader崩溃或者leader失去大多数的follower，这时Zookeeper进入恢复模式，恢复模式需要重新选举出一个新的leader，让所有的Server都恢复到一个正确的状态。Zookeeper的选举算法有两种：一种是基于basic paxos实现的，另外一种是基于fast paxos算法实现的。系统默认的选举算法为**fast paxos**。

1、Zookeeper选主流程(basic paxos)
（1）选举线程由当前Server发起选举的线程担任，其主要功能是对投票结果进行统计，并选出推荐的Server； 
（2）选举线程首先向所有Server发起一次询问(包括自己)； 
（3）选举线程收到回复后，验证是否是自己发起的询问(验证zxid是否一致)，然后获取对方的id(myid)，并存储到当前询问对象列表中，最后获取对方提议的leader相关信息(id,zxid)，并将这些信息存储到当次选举的投票记录表中； 
（4）收到所有Server回复以后，就计算出zxid最大的那个Server，并将这个Server相关信息设置成下一次要投票的Server； 
（5）线程将当前zxid最大的Server设置为当前Server要推荐的Leader，如果此时获胜的Server获得n/2 + 1的Server票数，设置当前推荐的leader为获胜的Server，将根据获胜的Server相关信息设置自己的状态，否则，继续这个过程，直到leader被选举出来。 通过流程分析我们可以得出：要使Leader获得多数Server的支持，则Server总数必须是奇数2n+1，且存活的Server的数目不得少于n+1. 每个Server启动后都会重复以上流程。在恢复模式下，如果是刚从崩溃状态恢复的或者刚启动的server还会从磁盘快照中恢复数据和会话信息，Zookeeper会记录事务日志并定期进行快照，方便在恢复时进行状态恢复。

![](/img/2019-1-22-Zookeeper概念、原理、应用场景/zookeeper是如何选取主leader的1.png)



2、Zookeeper选主流程(fast paxos)
fast paxos流程是在选举过程中，某Server首先向所有Server提议自己要成为leader，当其它Server收到提议以后，解决epoch和 zxid的冲突，并接受对方的提议，然后向对方发送接受提议完成的消息，重复这个流程，最后一定能选举出Leader。

![](/img/2019-1-22-Zookeeper概念、原理、应用场景/zookeeper是如何选取主leader的2.png)

## 18、Zookeeper同步流程

选完Leader以后，Zookeeper就进入状态同步过程。 
1、Leader等待server连接； 
2、Follower连接leader，将最大的zxid发送给leader； 
3、Leader根据follower的zxid确定同步点； 
4、完成同步后通知follower 已经成为uptodate状态； 
5、Follower收到uptodate消息后，又可以重新接受client的请求进行服务了。

![](/img/2019-1-22-Zookeeper概念、原理、应用场景/Zookeeper同步流程.png)



## 19、分布式通知和协调

对于系统调度来说：操作人员发送通知实际是通过控制台**改变某个节点的状态**，**然后Zookeeper将这些变化发送给注册了这个节点的watcher的所有客户端**。
对于执行情况汇报：每个工作进程都在某个目录下**创建一个临时节点**。**并携带工作的进度数据**，这样**汇总的进程可以监控目录子节点的变化获得工作进度的实时的全局情况**。

## 20、机器中为什么会有leader？

在分布式环境中，有些业务逻辑只需要集群中的某一台机器进行执行，**其他的机器可以共享这个结果**，这样可以大大**减少重复计算**，**提高性能**，于是就需要进行leader选举。

## 21、Zookeeper节点宕机如何处理？

Zookeeper本身也是集群，推荐配置不少于3个服务器。Zookeeper自身也要保证当一个节点宕机时，其他节点会继续提供服务。
如果是一个Follower宕机，还有2台服务器提供访问，因为Zookeeper上的数据是有多个副本的，数据并不会丢失；
如果是一个Leader宕机，Zookeeper会选举出新的Leader。
Zookeeper集群的机制是只要超过半数的节点正常，集群就能正常提供服务。只有在Zookeeper节点挂得太多，只剩一半或不到一半节点能工作，集群才失效。
所以
3个节点的cluster可以挂掉1个节点(leader可以得到2票>1.5)
2个节点的cluster就不能挂掉任何1个节点了(leader可以得到1票<=1)

## 22、Zookeeper负载均衡和nginx负载均衡区别

Zookeeper的负载均衡是可以调控，nginx只是能调权重，其他需要可控的都需要自己写插件；但是nginx的吞吐量比Zookeeper大很多，应该说按业务选择用哪种方式。

## 23、Zookeeper watch机制

Watch机制官方声明：一个Watch事件是一个一次性的触发器，当被设置了Watch的数据发生了改变的时候，则服务器将这个改变发送给设置了Watch的客户端，以便通知它们。
Zookeeper机制的特点：
1、一次性触发数据发生改变时，一个watcher event会被发送到client，但是client**只会收到一次这样的信息**。
2、watcher event异步发送watcher的通知事件从server发送到client是**异步**的，这就存在一个问题，不同的客户端和服务器之间通过socket进行通信，由于**网络延迟或其他因素导致客户端在不通的时刻监听到事件**，由于Zookeeper本身提供了**ordering guarantee，即客户端监听事件后，才会感知它所监视znode发生了变化**。所以我们使用Zookeeper不能期望能够监控到节点每次的变化。Zookeeper**只能保证最终的一致性，而无法保证强一致性**。
3、数据监视Zookeeper有数据监视和子数据监视getdata() and exists()设置数据监视，getchildren()设置了子节点监视。
4、注册watcher **getData、exists、getChildren**
5、触发watcher **create、delete、setData**
6、**setData()**会触发znode上设置的data watch（如果set成功的话）。一个成功的**create()** 操作会触发被创建的znode上的数据watch，以及其父节点上的child watch。而一个成功的**delete()**操作将会同时触发一个znode的data watch和child watch（因为这样就没有子节点了），同时也会触发其父节点的child watch。
7、当一个客户端**连接到一个新的服务器上**时，watch将会被以任意会话事件触发。当**与一个服务器失去连接**的时候，是无法接收到watch的。而当client**重新连接**时，如果需要的话，所有先前注册过的watch，都会被重新注册。通常这是完全透明的。只有在一个特殊情况下，**watch可能会丢失**：对于一个未创建的znode的exist watch，如果在客户端断开连接期间被创建了，并且随后在客户端连接上之前又删除了，这种情况下，这个watch事件可能会被丢失。
8、Watch是轻量级的，其实就是本地JVM的**Callback**，服务器端只是存了是否有设置了Watcher的布尔类型

## 24、Zookeeper使用场景

### 场景一：客户端动态感知服务端节点变化，实现高可用

现在假设有这样一种需求：服务端节点有多个，可以动态的上下线；需要让任意一台客户端都能实时感知服务端节点的变化，进而连接目前可提供服务的节点。 实现思路：我们可以借助于Zookeeper这个第三方中间件，在每台服务器启动时都向Zookeeper注册服务器的节点信息（比如：/servers/server01;/servers/server02）；客户端每次调用之前都通过getChildren方法获取最新的服务器节点信息，同时客户端在Zookeeper注册监听，监听服务器节点的变化；如果某刻服务器server01下线了，Zookeeper就会发出节点变化通知客户端，回调process方法拉取最新的服务器节点信息。 服务端代码如下：

```java
public class DistributeServer {

    private static final String CONNECT_URL = "mini1:2181,mini2:2181,mini3:2181";

    private static final int SESSION_TIME_OUT = 2000;

    private static final String PARENT_NODE = "/servers";

    private Zookeeper ZookeeperCli = null;

    /**
     * @Description 获取连接
     * @Author 刘俊重
     * @Date 2017/12/13
     */
    public void getConnect() throws Exception{
        ZookeeperCli = new Zookeeper(CONNECT_URL, SESSION_TIME_OUT, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                System.out.println(event.getType()+"-----------"+event.getPath());
                try{
                    ZookeeperCli.getChildren("/", true);
                }catch (Exception e){

                }

            }
        });
    }

    /**
     * @Description 服务器启动时向Zookeeper注册服务信息
     * @Author 刘俊重
     * @Date 2017/12/13
     */
    public void registerServer(String hostName) throws  Exception {
        String s = ZookeeperCli.create(PARENT_NODE + "/", hostName.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println("服务器："+hostName+"已经注册完毕");
    }

    /**
     * @Description 模拟实际的业务操作
     * @Author 刘俊重
     * @Date 2017/12/13
     */
    public void handelBusiness(String hostname) throws  Exception {
        System.out.println("服务器："+hostname+"正在处理业务。。。");
        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws  Exception {
        DistributeServer server = new DistributeServer();
        server.getConnect();
        server.registerServer(args[0]);
        server.handelBusiness(args[0]);
    }
}
```

客户端代码如下：

```java
/**
 * @author 刘俊重
 * @Description 模拟客户端，拉取最新服务器节点列表并向Zookeeper设置监听
 */
public class DistributeClient {

    private static final String CONNECT_URL = "mini1:2181,mini2:2181,mini3:2181";

    private static final int SESSION_TIME_OUT = 2000;

    private static final String PARENT_NODE = "/servers";

    private Zookeeper ZookeeperCli = null;

    private volatile List<String> serverList = null;

    /**
     * @Description 获取连接
     * @Author 刘俊重
     * @Date 2017/12/13
     */
    public void getConnect() throws Exception{
        ZookeeperCli = new Zookeeper(CONNECT_URL, SESSION_TIME_OUT, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                // 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
                System.out.println(event.getType()+"-----------"+event.getPath());
                try{
                    //重新更新服务器列表，并且注册了监听
                    getServerList();
                }catch (Exception e){

                }

            }
        });
    }

    /**
     * @Description 获取服务器子节点信息，并对父节点进行监听
     * @Author 刘俊重
     */
    public void getServerList() throws  Exception {
        List<String> children = ZookeeperCli.getChildren(PARENT_NODE, true);
        List<String> servers = new ArrayList<String>();
        for(String child : children){
            // child只是子节点的节点名
            byte[] data = ZookeeperCli.getData(PARENT_NODE + "/" + child, false, null);
            servers.add(new String(data));
        }
        //把servers赋值给成员变量serverList，以提供给各业务线程使用
        serverList = servers;
        System.out.println("节点数据："+serverList);

    }
    /**
     * @Description 模拟实际的业务操作
     * @Author 刘俊重
     * @Date 2017/12/13
     */
    public void handelBusiness() throws  Exception {
        System.out.println("客户端开始工作。。。");
        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws Exception {
        DistributeClient client = new DistributeClient();

        client.getConnect();
        client.getServerList();
        client.handelBusiness();
    }
}

```



### 使用场景二：分布式锁实现

假设现在集群中有50台机器对某台机器上的同一文件进行修改，如何才能保证这个文件不会被写乱呢，使用java中的synchronized锁肯定是不行的，因为这个锁是对某个程序而言的，而我们这根本就不是在一个服务器上，怎么会锁的住，用Zookeeper实现的分布式锁可以实现。 设计思路：服务器启动时都去Zookeeper上注册一个“短暂+序号”的znode节点（如/lock/1;/lock/2），并设置监听父节点变化；获取到父节点下所有子节点，并比较序号的大小；约定比如序号最小的获取锁，去操作某一文件，操作完成后删除自己的节点（相当于释放锁），并注册一个新的“短暂+序号”的znode节点；其它程序收到Zookeeper发送的节点变化的通知之后，去比较序号的大小，看谁获得新锁。

```java
public class DistributedClientLock {
    // 会话超时
    private static final int SESSION_TIMEOUT = 2000;
    // Zookeeper集群地址
    private String hosts = "mini1:2181,mini2:2181,mini3:2181";
    private String groupNode = "locks";
    private String subNode = "sub";
    private boolean haveLock = false;

    private Zookeeper Zookeeper;
    // 记录自己创建的子节点路径
    private volatile String thisPath;

    /**
     * 连接Zookeeper
     */
    public void connectZookeeper() throws Exception {
        Zookeeper = new Zookeeper(hosts, SESSION_TIMEOUT, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                try {

                    // 判断事件类型，此处只处理子节点变化事件
                    if (event.getType() == EventType.NodeChildrenChanged && event.getPath().equals("/" + groupNode)) {
                        //获取子节点，并对父节点进行监听
                        List<String> childrenNodes = Zookeeper.getChildren("/" + groupNode, true);
                        String thisNode = thisPath.substring(("/" + groupNode + "/").length());
                        // 去比较是否自己是最小id
                        Collections.sort(childrenNodes);
                        if (childrenNodes.indexOf(thisNode) == 0) {
                            //访问共享资源处理业务，并且在处理完成之后删除锁
                            doSomething();

                            //重新注册一把新的锁
                            thisPath = Zookeeper.create("/" + groupNode + "/" + subNode, null, Ids.OPEN_ACL_UNSAFE,
                                    CreateMode.EPHEMERAL_SEQUENTIAL);
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        // 1、程序一进来就先注册一把锁到Zookeeper上
        thisPath = Zookeeper.create("/" + groupNode + "/" + subNode, null, Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL);

        // wait一小会，便于观察
        Thread.sleep(new Random().nextInt(1000));

        // 从Zookeeper的锁父目录下，获取所有子节点，并且注册对父节点的监听
        List<String> childrenNodes = Zookeeper.getChildren("/" + groupNode, true);

        //如果争抢资源的程序就只有自己，则可以直接去访问共享资源
        if (childrenNodes.size() == 1) {
            doSomething();
            thisPath = Zookeeper.create("/" + groupNode + "/" + subNode, null, Ids.OPEN_ACL_UNSAFE,
                    CreateMode.EPHEMERAL_SEQUENTIAL);
        }
    }

    /**
     * 处理业务逻辑，并且在最后释放锁
     */
    private void doSomething() throws Exception {
        try {
            System.out.println("gain lock: " + thisPath);
            Thread.sleep(2000);
        } finally {
            System.out.println("finished: " + thisPath);
            //释放锁
            Zookeeper.delete(this.thisPath, -1);
        }
    }

    public static void main(String[] args) throws Exception {
        DistributedClientLock dl = new DistributedClientLock();
        dl.connectZookeeper();
        Thread.sleep(Long.MAX_VALUE);
    }

}
```

