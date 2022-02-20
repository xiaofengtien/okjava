## 从入门到入土（九）手摸手教你搭建RocketMQ双主双从同步集群，不信学不会！

原创 编程界的小学生 [Java知音](javascript:void(0);) *2020-07-13*

收录于话题

\#知音精选63

\#实战项目301

\#RocketMQ194

# 

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbucGRda0rcJFUcQBDSTWOLQwIxh0BtyOOiaibYXRzCjz4ID20aW2ZLKn18KekUCib3d8yLVtfH1tmljUQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

**精彩推荐**

[一百期Java面试题汇总](http://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247484532&idx=1&sn=1c243934507d79db4f76de8ed0e5727f&chksm=e80db202df7a3b14fe7077b0fe5ec4de4088ce96a2cde16cbac21214956bd6f2e8f51193ee2b&scene=21#wechat_redirect)

[SpringBoot内容聚合](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247486774&idx=3&sn=a13875e4a12fc7a32d39d6ac4ca15b33&chksm=fc7a6098cb0de98e56ee49654d2b31ea0309709166ebbf6d820fc41570bc6445b514b4940b6b&scene=21#wechat_redirect)

[IntelliJ IDEA内容聚合](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247486813&idx=2&sn=509ba858333ac30a2470b817cd6dc5e4&chksm=fc7a60f3cb0de9e5fb1b1f5d36239d97a82be70776020e4113a3dbba57b8afee5e3e218abc6e&scene=21#wechat_redirect)

[Mybatis内容聚合](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247486774&idx=2&sn=71fd7375bce8907e5d325b25199c9f8c&chksm=fc7a6098cb0de98e326777f8231be4ddc9c37d2610912e126fb4dc89c32627fcec60353cb638&scene=21#wechat_redirect)

#  

接上一篇：[从入门到入土（八）RocketMQ的Consumer是如何做的负载均衡的](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247494327&idx=2&sn=97fde4b7c346b9ef3a7ae6d421a6a786&chksm=ebd5d59bdca25c8d433d9598abd1327651d4636a2f35c679eb86da6ec22eed4bd3b16ab37767&scene=21#wechat_redirect)

# 一、环境准备 

## 1、补充

如果单机都不会安装的，或者管控台不会安装的请先前往如下这篇文章：

https://blog.csdn.net/ctwctw/article/details/107143968

> 再次强调，如果单机都不会的话，先抽出2min看看上面文章，因为需要改jvm配置的，默认8G，没那么大的内存启动会报错的。

## 2、机器

|     机器      |                   用途                   |
| :-----------: | :--------------------------------------: |
| 172.17.160.28 | namesrv、broker-a-master、broker-b-slave |
| 172.17.160.29 | namesrv、broker-b-master、broker-a-slave |

> 两台机器分别启动namesrv
>
> 172.17.160.28和172.17.160.29交叉作为彼此的slave

# 二、开始搭建

> 搭建2M2S SYNC。

## 1、说明

Rocket MQ在4.5.0之前版本支持的主从不支持自动切换，也就是并非真正意义上的高可用，比如2M2S，其中1个M挂了后，他的Slave节点不会自动升级为Master，但是在4.5.0开始迎来了Dledger这个“怪物”，他是基于raft的（Redis的哨兵也是这个玩意），让Rocket MQ真正意义的做到了高可用，Master挂了后Slave自动升级为Master，完全自动，释放双手。

## 2、古老的集群搭建

### 2.1、准备

进入到2m2s-sync目录

```
cd /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/conf/2m-2s-sync
```

### 2.2、Master1

> 也就是172.17.160.28机器上的broker-a-master节点。

修改其配置文件然后启动即可。

```
vi broker-a.properties
```

修改内容如下：

```
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，需要注意的是slave节点需要和master节点的brokerName一致，区分m还是s交由下面的brokerId来配置。
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=172.17.160.28:9876;172.17.160.29:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#存储路径
storePathRootDir=/home/chentongwei/data/rocketmq/broker-a/store
#commitLog 存储路径
storePathCommitLog=/home/chentongwei/data/rocketmq/broker-a/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/home/chentongwei/data/rocketmq/broker-a/store/consumequeue
#消息索引存储路径
storePathIndex=/home/chentongwei/data/rocketmq/broker-a/store/index
#checkpoint 文件存储路径
storeCheckpoint=/home/chentongwei/data/rocketmq/broker-a/store/checkpoint
#abort 文件存储路径
abortFile=/home/chentongwei/data/rocketmq/broker-a/store/abort
#限制的消息大小
maxMessageSize=65536
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
```

### 2.3、Slave1

> 也就是172.17.160.29机器上的broker-a-slave节点。

修改其配置文件然后启动即可。

```
vi broker-a-s.properties
```

修改内容如下：

```
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，需要注意的是slave节点需要和master节点的brokerName一致，区分m还是s交由下面的brokerId来配置。
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
# 注意此处是1了，不在是0了，因为0代表Master
brokerId=1
#nameServer地址，分号分割
namesrvAddr=172.17.160.28:9876;172.17.160.29:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=11911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#存储路径
storePathRootDir=/home/chentongwei/data/rocketmq/broker-a/store
#commitLog 存储路径
storePathCommitLog=/home/chentongwei/data/rocketmq/broker-a/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/home/chentongwei/data/rocketmq/broker-a/store/consumequeue
#消息索引存储路径
storePathIndex=/home/chentongwei/data/rocketmq/broker-a/store/index
#checkpoint 文件存储路径
storeCheckpoint=/home/chentongwei/data/rocketmq/broker-a/store/checkpoint
#abort 文件存储路径
abortFile=/home/chentongwei/data/rocketmq/broker-a/store/abort
#限制的消息大小
maxMessageSize=65536
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
```

### 2.4、Master2

> 也就是172.17.160.29机器上的broker-b-master节点。

修改其配置文件然后启动即可。

```
vi broker-b.properties
```

修改内容如下：

```
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，需要注意的是slave节点需要和master节点的brokerName一致，区分m还是s交由下面的brokerId来配置。
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=172.17.160.28:9876;172.17.160.29:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#存储路径
storePathRootDir=/home/chentongwei/data/rocketmq/broker-b/store
#commitLog 存储路径
storePathCommitLog=/home/chentongwei/data/rocketmq/broker-b/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/home/chentongwei/data/rocketmq/broker-b/store/consumequeue
#消息索引存储路径
storePathIndex=/home/chentongwei/data/rocketmq/broker-b/store/index
#checkpoint 文件存储路径
storeCheckpoint=/home/chentongwei/data/rocketmq/broker-b/store/checkpoint
#abort 文件存储路径
abortFile=/home/chentongwei/data/rocketmq/broker-b/store/abort
#限制的消息大小
maxMessageSize=65536
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
```

### 2.5、Slave2

> 也就是172.17.160.28机器上的broker-b-slave节点。

修改其配置文件然后启动即可。

```
vi broker-b-s.properties
```

修改内容如下：

```
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，需要注意的是slave节点需要和master节点的brokerName一致，区分m还是s交由下面的brokerId来配置。
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
# 注意此处是1了，不在是0了，因为0代表Master
brokerId=1
#nameServer地址，分号分割
namesrvAddr=172.17.160.28:9876;172.17.160.29:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=11911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#存储路径
storePathRootDir=/home/chentongwei/data/rocketmq/broker-b/store
#commitLog 存储路径
storePathCommitLog=/home/chentongwei/data/rocketmq/broker-b/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/home/chentongwei/data/rocketmq/broker-b/store/consumequeue
#消息索引存储路径
storePathIndex=/home/chentongwei/data/rocketmq/broker-b/store/index
#checkpoint 文件存储路径
storeCheckpoint=/home/chentongwei/data/rocketmq/broker-b/store/checkpoint
#abort 文件存储路径
abortFile=/home/chentongwei/data/rocketmq/broker-b/store/abort
#限制的消息大小
maxMessageSize=65536
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
```

### 2.6、总结

其实就是如下几个关键配置需要注意：

- brokerClusterName：集群名称，所有broker配置的必须一致
- brokerName：broker名称，M-S之间必须一致，比如broker-a/broker-a（M/S），broker-b/broker-b
- brokerId：0 表示 Master，>0 表示 Slave，比如1M2S，那么M是0，Slave1是1，Slave2配置为2
- namesrvAddr：namesrv地址
- store*：持久化数据存储目录
- brokerRole：Broker的角色
- flushDiskType：刷盘方式，2m2s-sync中，master默认为SYNC_FLUSH，slave默认为ASYNC_FLUSH

### 2.7、启动

#### 2.7.1、启动namesrv

```
cd /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/bin
nohup sh mqnamesrv &
```

> 两台机器都要启动。

#### 2.7.2、启动broker

*172.17.160.28*

```
cd /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/bin
# 启动Master1
nohup sh mqbroker -c /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/conf/2m-2s-sync/broker-a.properties &

# 启动Slave2
nohup sh mqbroker -c /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/conf/2m-2s-sync/broker-b-s.properties &
```

*172.17.160.29*

```
cd /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/bin
# 启动Master2
nohup sh mqbroker -c /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/conf/2m-2s-sync/broker-b.properties &

# 启动Slave1
nohup sh mqbroker -c /home/chentongwei/rocketmq/rocketmq-all-4.7.0-bin-release/conf/2m-2s-sync/broker-a-s.properties &
```

### 2.8、验证

直接打开我们的管控台进行查看集群（Cluster）

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufsgPtTnEa4FfQFllDddM1Cc3LNJvqQcWdKPH8MbNwxLPmaVzvicjKF9HuRuKRH9kMjWj1tXBuMjEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.9、问题

他致命的问题就是不能自动主备切换，比如我杀死broker-a-master的进程，然后看管控台broker-a-slave是否会自动升级为master，发现并不会。所以我们的Dledger诞生了。

```
# 找到broker-a-master的进程号是8983
kill -9 8983
```



![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufsgPtTnEa4FfQFllDddM1CA0lH0wTBQbNXgAg54COfDGFklpKl0XqoDEwRWofTebS6LzaLFrLSCA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 3、Dledger搭建

### 3.1、说明

是基于raft的一种做法，redis的哨兵也是基于raft的，和这个是一样的道理。上面【古老的集群搭建】方式并不作废，Dledger就是基于上面的继续增加额外配置，上面的配置依然保留，继续追加新的配置即可。Dledger要求机器数必须是3台或3台以上才行。这也是raft的基础要求。不懂的可以自行Google下raft。

### 3.2、搭建

#### 3.2.1、官方搭建手册

https://github.com/apache/rocketmq/blob/master/docs/cn/dledger/deploy_guide.md

#### 3.2.2、准备

我们多开一台虚拟机：172.17.160.30，采取Dledger搭建一组1M2S的集群，机器规划如下：

|     机器      |           用途           |
| :-----------: | :----------------------: |
| 172.17.160.28 | namesrv、broker-a-master |
| 172.17.160.29 | namesrv、broker-a-slave1 |
| 172.17.160.30 | namesrv、broker-a-slave2 |

#### 3.2.3、配置

> **上面的【古老的集群搭建】配置不变，在每个properties里新增如下配置：**

*172.17.160.28*

```
enableDLegerCommitLog=true
dLegerGroup=broker-a
dLegerPeers=n0-172.17.160.28:40911;n1-172.17.160.29:40911;n2-172.17.160.30:40911
dLegerSelfId=n0
sendMessageThreadPoolNums=4
```

*172.17.160.29*

```
enableDLegerCommitLog=true
dLegerGroup=broker-a
dLegerPeers=n0-172.17.160.28:40911;n1-172.17.160.29:40911;n2-172.17.160.30:40911
dLegerSelfId=n1
sendMessageThreadPoolNums=4
```

*172.17.160.30*

```
enableDLegerCommitLog=true
dLegerGroup=broker-a
dLegerPeers=n0-172.17.160.28:40911;n1-172.17.160.29:40911;n2-172.17.160.30:40911
dLegerSelfId=n2
sendMessageThreadPoolNums=4
```

#### 3.2.4、配置说明

| name                        | 含义                                                         | 举例                                                         |
| :-------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| enableDLegerCommitLog       | 是否启动 DLedger                                             | true                                                         |
| dLegerGroup                 | DLedger Raft Group的名字，建议和 brokerName 保持一致         | broker-a                                                     |
| dLegerPeers                 | DLedger Group 内各节点的端口信息，同一个 Group 内的各个节点配置必须要保证一致 | n0-172.17.160.28:40911;n1-172.17.160.29:40911;n2-172.17.160.30:40911 |
| dLegerSelfId                | 节点 id, 必须属于 dLegerPeers 中的一个；同 Group 内各个节点要唯一 | n0                                                           |
| sendMessageThreadPoolNums=4 | 发送消息的线程数，建议与CPU核数设置成一样的                  | 4                                                            |

#### 3.2.5、启动

启动三台的namesrv和三个broker，跟上面姿势一样，不重复贴代码了。

#### 3.2.6、验证

管控台去看的话应该是1个master，两个slave，都叫broker-a，这时候停掉master，会发现其中一个slave自动升级为Master了，这才叫真正的高可用，自动主备切换。

### 3.3、补充

虽然要求至少三台机器，但是没条件的可以用一台机器去模拟，弄三个不同的端口号就行了。

# 三、常见问题

## 1、Lock failed,MQ already started

### 1.1、错误

```
[root@iZ2ze84zygpzjw5bfcmh2hZ 2m-2s-sync]#  sh ../../bin/mqbroker -c broker-b-s.properties
java.lang.RuntimeException: Lock failed,MQ already started
        at org.apache.rocketmq.store.DefaultMessageStore.start(DefaultMessageStore.java:223)
        at org.apache.rocketmq.broker.BrokerController.start(BrokerController.java:853)
        at org.apache.rocketmq.broker.BrokerStartup.start(BrokerStartup.java:64)
        at org.apache.rocketmq.broker.BrokerStartup.main(BrokerStartup.java:58)
```

### 1.2、解决

这个是由于没配置store*导致的，比如如下：

```
#存储路径
storePathRootDir=/home/chentongwei/data/rocketmq/broker-a/store
#commitLog 存储路径
storePathCommitLog=/home/chentongwei/data/rocketmq/broker-a/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/home/chentongwei/data/rocketmq/broker-a/store/consumequeue
#消息索引存储路径
storePathIndex=/home/chentongwei/data/rocketmq/broker-a/store/index
#checkpoint 文件存储路径
storeCheckpoint=/home/chentongwei/data/rocketmq/broker-a/store/checkpoint
#abort 文件存储路径
abortFile=/home/chentongwei/data/rocketmq/broker-a/store/abort
```

没配置这个，我一台机器上部署两个broker，一个broker-a-master，一个broker-b-slave，都采取默认的store*配置，冲突了，所以报错。所以手动配上不同目录即可，我这里目录区分是broker-a/broker-b。

## 2、AllocateMappedFileService started:false lastThread:null

这个是由于store*配置的都一样导致的，比如如下：

```
#存储路径
storePathRootDir=/home/chentongwei/data/rocketmq/broker-a/store
#commitLog 存储路径
storePathCommitLog=/home/chentongwei/data/rocketmq/broker-a/store
#消费队列存储路径存储路径
storePathConsumeQueue=/home/chentongwei/data/rocketmq/broker-a/store
#消息索引存储路径
storePathIndex=/home/chentongwei/data/rocketmq/broker-a/store
#checkpoint 文件存储路径
storeCheckpoint=/home/chentongwei/data/rocketmq/broker-a/store
#abort 文件存储路径
abortFile=/home/chentongwei/data/rocketmq/broker-a/store
```

发现了没？我都用的`/home/chentongwei/data/rocketmq/broker-a/store`，这肯定不行的，所以我们需要采用不同目录，比如`store/commitlog`、`store/consumequeue`等。

## 3、Address already in use

这个更神奇了，见名知意，地址被占用了。也就是端口被占用了。我为什么说他更神奇，不是因为我没配置listenPort，而是我在单台机器上配置了broker-a-master的端口为默认端口10911，broker-b-slave的端口为10912，就是+1操作了。他为啥说我被占用？

经过查源码发现，他大爷的，他给我默认启动了一个进程，端口号就是配置的端口+1，所以启动一个broker后他其实是给我们启动了两个端口，一个原端口，一个原端口+1。所以我配置的时候一个是10911，一个是11911，而不能 是10912！！！

```
#Broker 对外服务的监听端口
listenPort=11911
```

源码如下：

```
// {@link org.apache.rocketmq.broker.BrokerStartup#createBrokerController(String[])}
messageStoreConfig.setHaListenPort(nettyServerConfig.getListenPort() + 1);
```

## 4、管控台配置namesrv

如果namesrv和broker都启动正常后，管控台上还是没有集群的话，可以检查下管控台是否配置了namesrv



![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufsgPtTnEa4FfQFllDddM1CdIIic1p8uN4DrDskvqkNclpBiaIVu3gjeERKxTngWY53OeUpraEMWIOQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**这个namesrv配置需要手动敲完后+回车+点击UPDATE才能生效。**

**END**