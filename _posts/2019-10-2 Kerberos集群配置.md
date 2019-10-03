---
layout:     post
title:      Kerberos集群配置
subtitle:   
date:       2019-10-2
author:     BY KiloMeter
header-img: img/2019-10-2 Kerberos集群配置/68.png
catalog: true
tags:
    - Kerberos
---

在实习的时候接触到了kerberos，按照网上的博客搞了好久才把kerberos调通，调通后续还是遇到了许多关于kerberos的问题，在这里我使用本机的三台虚拟机，搭了个开启kerberos的kafka，把整个流程给记录下，以后如果还有需要自己搭建kerberos的话，有个参考~。

整个配置流程：

首先在其中某台机器上安装kdc，其他几台机器安装kerberos客户端即可，安装过程这里不写了。

可以参考[Kerberos部署,配置与基础使用]([https://leibnizhu.gitlab.io/2018/03/07/Kerberos%E9%83%A8%E7%BD%B2-%E9%85%8D%E7%BD%AE%E4%B8%8E%E5%9F%BA%E7%A1%80%E4%BD%BF%E7%94%A8/](https://leibnizhu.gitlab.io/2018/03/07/Kerberos部署-配置与基础使用/))进行安装。

接下来，修改/etc/krb.conf文件，并将修改后的文件分发到其他机器的/etc目录下

```properties
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
# 默认使用的realm是HADOOP.COM，如果想要修改的话，下面的realms也需要对应修改，我这里就使用默认的
 default_realm = HADOOP.COM
 dns_lookup_realm = false
 ticket_lifetime = 500d
 renew_lifetime = 500d
 forwardable = true
 rdns = false
 default_ccache_name = KEYRING:persistent:%{uid}
# 这部分需要指定kdc所在的机器
[realms]
 HADOOP.COM = {
  kdc = ambari1
  admin_server = ambari1
 }

```

创建kerberos数据库，存放账号信息，可以看上面的那篇博客进行创建，创建后需要为每台机器添加账号，使用addprinc 用户名@域名依次进行创建，创建完之后，使用xst -k /path/to/*.keytab -norandkey 用户名@域名，依次把上面三个账号的keytab文件导出来，然后分发到对应的机器上去(所有配置文件最好都放在所有机器的相同位置上)，使用kinit 用户名@域名 -k -t 对应keytab文件进行登录。然后，开始修改kafka相关的配置文件，首先修改KAFKA_HOME/config/server.properties文件，添加以下内容

```properties
# 这里的ambari2指的是本机的主机名，每台机器的配置文件中，这个listeners需要修改成自己的主机名
listeners=SASL_PLAINTEXT://ambari2:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=GSSAPI
sasl.enabled.mechanisms=GSSAPI
#这里的name需要修改成上面创建账号时使用的用户名，我这里创建的用户名都统一为kafka
sasl.kerberos.service.name=kafka
```
 
之后需要修改kafka的JVM启动参数，添加kerberos的相关变量，如下。

![](/img/2019-10-2 Kerberos集群配置/添加kafka的JVM参数.png)

到这里就配置完成了，接下来重启下kafka即可。

在所有配置都配置完之后，在某一台broker上开启生产者程序，在本地上开启消费者程序，生产者生产数据后报如下警告

![](/img/2019-10-2 Kerberos集群配置/配置完成后生产者警告消息.png)

这种情况似乎是开启kerberos后，开启生产者的命令需要添加一些其他的参数，暂时还未研究，但这个警告不影响kafka服务的使用。

在本地想要连接开启了kerberos的kafka的话，首先需要把/etc/krb.con，kafka_server_jaas.conf、kafka.keytab文件拷贝到本地，然后修改kafka_server_jaas.con文件，把里面的keytab文件路径修改成本地keytab文件的路径。

在程序种需要添加下面的参数，跟上面最后一步一样，需要添加JVM参数，

```java
System.setProperty("java.security.krb5.conf", LoadProperties.getValue("java.security.krb5.conf"));
        System.setProperty("java.security.auth.login.config", LoadProperties.getValue("java.security.auth.login.config"));
```

下面这个是上面读取的配置文件中对应的内容，就是拷贝到本地后的配置文件路径。

```properties
java.security.auth.login.config=D:/config/test/kafka_server_jaas.conf
java.security.krb5.conf=D:/config/test/krb5.conf
```

消费者除了之前的key.deserializer等参数外，还需要下面这三个参数

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=GSSAPI
sasl.kerberos.service.name=kafka
```

这三个参数和前面在修改server.properties中也有出现过，保持一样即可。

本地开启消费者程序报错

![](/img/2019-10-2 Kerberos集群配置/本地开启消费者程序报错.png)

解决方法：

查看了下kafka_server_jaas.conf，发现keytab文件路径写错了，改过来即可。

最后在本地开启生产者消费者程序能够正常生产和消费，完成。

下面记录下kerberos常用的几个命令：

| 功能               | 命令                                                         |
| ------------------ | ------------------------------------------------------------ |
| 进入kadmin         | kadmin.local/kadmin                                          |
| 查看KeyTab         | klist -e -k -t *.keytab                                      |
| 清除缓存           | kdestroy                                                     |
| 测试keytab可用性   | kinit -k -t /var/kerberos/krb5kdc/keytab/root.keytab root/master1@JENKIN.COM |
| **kadmin模式**     |                                                              |
| 查看principal      | listprincs                                                   |
| 添加/删除principle | addprinc/delprinc admin/admin                                |
| 为各实例生成密钥   | xst -k xxxx.keytab xxxx@xxxx.COM                             |



