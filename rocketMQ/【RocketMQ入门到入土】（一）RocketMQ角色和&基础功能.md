# 一、RocketMQ的安装

## 1、文档

**官方网站**

http://rocketmq.apache.org

**GitHub**

https://github.com/apache/rocketmq

## 2、下载

```
wget https://mirror.bit.edu.cn/apache/rocketmq/4.7.0/rocketmq-all-4.7.0-bin-release.zip
```

我们是基于Centos8来的，面向官方文档学习，所以下载地址自然也是官方的。

去官方网站找合适的版本进行下载，目前我这里最新的是4.7.0版本。

http://rocketmq.apache.org/dowloading/releases/

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4uCSdaxjiaCdicsquYrkZhaZbVWAO5bGreZBGdUosbWic0jAZmvEjQ7N2w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.7.0/rocketmq-all-4.7.0-bin-release.zip

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4MXicic8JyPpWjBNGC9nsmuTq0ouUducFf6bhGXoZMsODYxnJWbWK5Whg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 3、准备工作

### 3.1、解压

```
unzip rocketmq-all-4.7.0-bin-release.zip
```

### 3.2、安装jdk

```
sudo yum install java-1.8.0-openjdk-devel
```

## 4、启动

### 4.1、启动namesrv

```
cd rocketmq-all-4.7.0-bin-release/bin
./mqnamesrv
```

### 4.2、启动broker

```
cd rocketmq-all-4.7.0-bin-release/bin
./mqbroker -n localhost:9876
```

常见错误以及解决方案：

常见错误：启动broker失败 `Cannot allocate memory`

```
[root@node-113b bin]# ./mqbroker -n localhost:9876
Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00000005c0000000, 8589934592, 0) failed
; error='Cannot allocate memory' (errno=12)#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 8589934592 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /usr/local/rocketmq/bin/hs_err_pid1997.log
```

解决方案：

是由于默认内存分配的太大了，超出了本机内存，直接OOM了。

修改bin/目录下的如下两个脚本

```
runbroker.sh
runserver.sh
```

在这两个脚本里都搜索`-server -Xms`，将其内存分配小点，自己玩的话512MB就足够了，够够的了！

### 4.3、启动成功标识

namesrv启动成功标识：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4N4WV5c0fvSQN6cJGBLbTHqvjgjLdoXialZl6gHyp91ajmAGG3aMyaUw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

broker启动成功标识：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4ktGEo4DBmibLUJzwvQ1ol4xeQkyKzLPUahjzPa6LgAPHQJWexAcL0Qg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 二、RocketMQ控制台的安装

控制台目前获取方式有如下两种：

1. 第三方网站去下载现成的，比如csdn等。
2. 官方源码包自己编译而成，官方没有现成的。

我们这里当然采取官方方式。

## 1、官方文档

**github仓库**

https://github.com/apache/rocketmq-externals

**中文指南**

https://github.com/apache/rocketmq-externals/blob/master/rocketmq-console/doc/1_0_0/UserGuide_CN.md

## 2、下载源码

https://codeload.github.com/apache/rocketmq-externals/zip/master

## 3、修改配置（可选）

我们下载完解压后的文件目录如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4Xic5ibuzbgzbzzyl1JAclBic0lU4J9vYJJ4Y8IFLcFu6HZf3fuCjuhs1A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

修改`rocketmq-console\src\main\resources\application.properties`文件的`server.port`就欧了。默认8080。

## 4、编译打包

进入`rocketmq-console`，然后用maven进行编译打包

```
mvn clean package -DskipTests
```

打包完会在target下生成我们spring boot的jar程序，直接`java -jar`启动完事。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4wY6Q8NXxeO8Q1iaHu1d6Rm8ysCL24I9L2sStrbyWaf5eTXTo6ic88yJw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 5、启动控制台

将编译打包好的springboot程序扔到服务器上，执行如下命令进行启动

```
java -jar rocketmq-console-ng-1.0.1.jar --rocketmq.config.namesrvAddr=127.0.0.1:9876
```

> 如果想后台启动就nohup &

访问一下看看效果：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4pdI11D3ytgeHdrE8aKPsuWiaYE3LJa5ehB3rHPTqzzzMHanI1p8XsnQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 三、测试

> rocketmq给我们提供了测试工具和测试类，可以在安装完很方便的进行测试。

## 0、准备工作

rocketmq给我们提供的默认测试工具在bin目录下，叫`tools.sh`。我们测试前需要配置这个脚本，为他指定namesrv地址才可以，否则测试发送/消费消息的时候会出现如下错误 **connect to null failed**：

```
22:49:02.470 [main] DEBUG i.n.u.i.l.InternalLoggerFactory - Using SLF4J as the default logging framework
RocketMQLog:WARN No appenders could be found for logger (io.netty.util.internal.PlatformDependent0).
RocketMQLog:WARN Please initialize the logger system properly.
java.lang.IllegalStateException: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to null failed
```

配置如下：

```
vim tools.sh
# 在export JAVA_HOME上面添加如下这段代码
export NAMESRV_ADDR=localhost:9876
```

## 1、发送消息

```
./tools.sh org.apache.rocketmq.example.quickstart.Producer
```

> 成功的话会看到哗哗哗的日志，因为这个类会发送1000条消息到TopicTest这个Topic下。

## 2、消费消息

```
./tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

> 成功的话会看到哗哗哗的日志，因为这个类会消费TopicTest下的全部消息。刚发送的1000条都会被消费掉。

## 3、控制台

发送成功后我们自然也能来到管控台去看消息和消费情况等等等信息

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4GOU9Ryf8AicJdLJFBESSqILhwicC9EXgwE3XONWQZnBZwI72PHxQzBsQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



# 四、架构图以及角色

## 1、架构图

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4O1wwicoI6FILy3fVuLPbef5HdDH2oIUKYzHyiaJdAYmtwiaw3UrO2YbrA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 2、角色

### 2.1、Broker

- 理解成RocketMQ本身
- broker主要用于producer和consumer接收和发送消息
- broker会定时向nameserver提交自己的信息
- 是消息中间件的消息存储、转发服务器
- 每个Broker节点，在启动时，都会遍历NameServer列表，与每个NameServer建立长连接，注册自己的信息，之后定时上报

### 2.2、Nameserver

- 理解成zookeeper的效果，只是他没用zk，而是自己写了个nameserver来替代zk
- 底层由netty实现，提供了路由管理、服务注册、服务发现的功能，是一个无状态节点
- nameserver是服务发现者，集群中各个角色（producer、broker、consumer等）都需要定时向nameserver上报自己的状态，以便互相发现彼此，超时不上报的话，nameserver会把它从列表中剔除
- nameserver可以部署多个，当多个nameserver存在的时候，其他角色同时向他们上报信息，以保证高可用，
- NameServer集群间互不通信，没有主备的概念
- nameserver内存式存储，nameserver中的broker、topic等信息默认不会持久化，所以他是无状态节点

### 2.3、Producer

- 消息的生产者
- 随机选择其中一个NameServer节点建立长连接，获得Topic路由信息（包括topic下的queue，这些queue分布在哪些broker上等等）
- 接下来向提供topic服务的master建立长连接（因为rocketmq只有master才能写消息），且定时向master发送心跳

### 2.4、Consumer

- 消息的消费者
- 通过NameServer集群获得Topic的路由信息，连接到对应的Broker上消费消息
- 由于Master和Slave都可以读取消息，因此Consumer会与Master和Slave都建立连接进行消费消息

## 3、核心流程

- Broker都注册到Nameserver上
- Producer发消息的时候会从Nameserver上获取发消息的topic信息
- Producer向提供服务的所有master建立长连接，且定时向master发送心跳
- Consumer通过NameServer集群获得Topic的路由信息
- Consumer会与所有的Master和所有的Slave都建立连接进行监听新消息

# 五、核心概念

## 1、Message

消息载体。Message发送或者消费的时候必须指定Topic。Message有一个可选的Tag项用于过滤消息，还可以添加额外的键值对。

## 2、topic

消息的逻辑分类，发消息之前必须要指定一个topic才能发，就是将这条消息发送到这个topic上。消费消息的时候指定这个topic进行消费。就是逻辑分类。

## 3、queue

1个Topic会被分为N个Queue，数量是可配置的。message本身其实是存储到queue上的，消费者消费的也是queue上的消息。多说一嘴，比如1个topic4个queue，有5个Consumer都在消费这个topic，那么会有一个consumer浪费掉了，因为负载均衡策略，每个consumer消费1个queue，5>4，溢出1个，这个会不工作。

## 4、Tag

Tag 是 Topic 的进一步细分，顾名思义，标签。每个发送的时候消息都能打tag，消费的时候可以根据tag进行过滤，选择性消费。

## 5、Message Model

消息模型：集群（Clustering）和广播（Broadcasting）

## 6、Message Order

消息顺序：顺序（Orderly）和并发（Concurrently）

## 7、Producer Group

消息生产者组

## 8、Consumer Group

消息消费者组

# 六、ACK

首先要明确一点：**ACK机制是发生在Consumer端的，不是在Producer端的**。也就是说Consumer消费完消息后要进行ACK确认，如果未确认则代表是消费失败，这时候Broker会进行重试策略（仅集群模式会重试）。ACK的意思就是：Consumer说：ok，我消费成功了。这条消息给我标记成已消费吧。

# 七、消费模式

## 1、集群模式（Clustering）

### 1.1、图解

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY42NN82MZKvy4YL89CPZzgfPH7LkotDc2nhuEO4n5x6APpnnzcExYLibw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### 1.2、特点

- 每条消息只需要被处理一次，broker只会把消息发送给消费集群中的一个消费者
- 在消息重投时，不能保证路由到同一台机器上
- 消费状态由broker维护

## 2、广播模式（Broadcasting）

### 2.1、图解

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4uCaC9aIuVOy4PMMFQDWr28SfJzT5TR6NicLarLyFq64C3yiaDj25JWEQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### 2.2、特点

- 消费进度由consumer维护
- 保证每个消费者都消费一次消息
- 消费失败的消息不会重投

# 八、Java API

说明：

- RocketMQ服务端版本为目前最新版：4.7.0
- Java客户端版本采取的目前最新版：4.7.0

pom如下

```
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.7.0</version>
</dependency>
```

## 1、Producer

> 发消息肯定要必备如下几个条件：
>
> - 指定生产组名（不能用默认的，会报错）
> - 配置namesrv地址（必须）
> - 指定topic name（必须）
> - 指定tag/key（可选）
>
> 验证消息是否发送成功：消息发送完后可以启动消费者进行消费，也可以去管控台上看消息是否存在。

### 1.1、send（同步）

```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 指定生产组名为my-producer
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        // 配置namesrv地址
        producer.setNamesrvAddr("124.57.180.156:9876");
        // 启动Producer
        producer.start();
        // 创建消息对象，topic为：myTopic001，消息内容为：hello world
        Message msg = new Message("myTopic001", "hello world".getBytes());
        // 发送消息到mq，同步的
        SendResult result = producer.send(msg);
        System.out.println("发送消息成功！result is : " + result);
        // 关闭Producer
        producer.shutdown();
        System.out.println("生产者 shutdown！");
    }
}
```

输出结果：

```
发送消息成功！result is : SendResult [sendStatus=SEND_OK, msgId=A9FE854140F418B4AAC26F7973910000, offsetMsgId=7B39B49D00002A9F00000000000589BE, messageQueue=MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=0], queueOffset=7]
生产者 shutdown！
```

### 1.2、send（批量）

```java
public class ProducerMultiMsg {
    public static void main(String[] args) throws Exception {
        // 指定生产组名为my-producer
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        // 配置namesrv地址
        producer.setNamesrvAddr("124.57.180.156:9876");
        // 启动Producer
        producer.start();

        String topic = "myTopic001";
        // 创建消息对象，topic为：myTopic001，消息内容为：hello world1/2/3
        Message msg1 = new Message(topic, "hello world1".getBytes());
        Message msg2 = new Message(topic, "hello world2".getBytes());
        Message msg3 = new Message(topic, "hello world3".getBytes());
        // 创建消息对象的集合，用于批量发送
        List<Message> msgs = new ArrayList<>();
        msgs.add(msg1);
        msgs.add(msg2);
        msgs.add(msg3);
        // 批量发送的api的也是send()，只是他的重载方法支持List<Message>，同样是同步发送。
        SendResult result = producer.send(msgs);
        System.out.println("发送消息成功！result is : " + result);
        // 关闭Producer
        producer.shutdown();
        System.out.println("生产者 shutdown！");
    }
}
```

输出结果：

```java
发送消息成功！result is : SendResult [sendStatus=SEND_OK, msgId=A9FE854139C418B4AAC26F7D13770000,A9FE854139C418B4AAC26F7D13770001,A9FE854139C418B4AAC26F7D13770002, offsetMsgId=7B39B49D00002A9F0000000000058A62,7B39B49D00002A9F0000000000058B07,7B39B49D00002A9F0000000000058BAC, messageQueue=MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=0], queueOffset=8]
生产者 shutdown！
```

> 从结果中可以看到只有一个msgId，所以可以发现虽然是三条消息对象，但是却只发送了一次，大大节省了client与server的开销。

错误情况：

批量发送的topic必须是同一个，如果message对象指定不同的topic，那么批量发送的时候会报错：

```java
Exception in thread "main" org.apache.rocketmq.client.exception.MQClientException: Failed to initiate the MessageBatch
For more information, please visit the url, http://rocketmq.apache.org/docs/faq/
    at org.apache.rocketmq.client.producer.DefaultMQProducer.batch(DefaultMQProducer.java:950)
    at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:898)
    at com.chentongwei.mq.rocketmq.ProducerMultiMsg.main(ProducerMultiMsg.java:29)
Caused by: java.lang.UnsupportedOperationException: The topic of the messages in one batch should be the same
    at org.apache.rocketmq.common.message.MessageBatch.generateFromList(MessageBatch.java:58)
    at org.apache.rocketmq.client.producer.DefaultMQProducer.batch(DefaultMQProducer.java:942)
    ... 2 more
```

### 1.3、sendCallBack（异步）

```java
public class ProducerASync {
    public static void main(String[] args) throws Exception {
       // 指定生产组名为my-producer
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        // 配置namesrv地址
        producer.setNamesrvAddr("124.57.180.156:9876");
        // 启动Producer
        producer.start();

        // 创建消息对象，topic为：myTopic001，消息内容为：hello world async
        Message msg = new Message("myTopic001", "hello world async".getBytes());
        // 进行异步发送，通过SendCallback接口来得知发送的结果
        producer.send(msg, new SendCallback() {
            // 发送成功的回调接口
            @Override
            public void onSuccess(SendResult sendResult) {
                System.out.println("发送消息成功！result is : " + sendResult);
            }
            // 发送失败的回调接口
            @Override
            public void onException(Throwable throwable) {
                throwable.printStackTrace();
                System.out.println("发送消息失败！result is : " + throwable.getMessage());
            }
        });

        producer.shutdown();
        System.out.println("生产者 shutdown！");
    }
}
```

输出结果：

```java
生产者 shutdown！
java.lang.IllegalStateException: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to [124.57.180.156:9876] failed
    at org.apache.rocketmq.client.impl.factory.MQClientInstance.updateTopicRouteInfoFromNameServer(MQClientInstance.java:681)
    at org.apache.rocketmq.client.impl.factory.MQClientInstance.updateTopicRouteInfoFromNameServer(MQClientInstance.java:511)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.tryToFindTopicPublishInfo(DefaultMQProducerImpl.java:692)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:556)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.access$300(DefaultMQProducerImpl.java:97)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl$4.run(DefaultMQProducerImpl.java:510)
    at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
Caused by: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to [124.57.180.156:9876] failed
    at org.apache.rocketmq.remoting.netty.NettyRemotingClient.getAndCreateNameserverChannel(NettyRemotingClient.java:441)
    at org.apache.rocketmq.remoting.netty.NettyRemotingClient.getAndCreateChannel(NettyRemotingClient.java:396)
    at org.apache.rocketmq.remoting.netty.NettyRemotingClient.invokeSync(NettyRemotingClient.java:365)
    at org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1371)
    at org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1361)
    at org.apache.rocketmq.client.impl.factory.MQClientInstance.updateTopicRouteInfoFromNameServer(MQClientInstance.java:624)
    ... 10 more
发送消息失败！result is : org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to [124.57.180.156:9876] failed
```

为啥报错了？很简单，他是异步的，从结果就能看出来，由于是异步的，我还没发送到mq呢，你就先给我shutdown了。肯定不行，所以我们在shutdown前面sleep 1s在看效果

```java
public class ProducerASync {
    public static void main(String[] args) throws Exception {
       // 指定生产组名为my-producer
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        // 配置namesrv地址
        producer.setNamesrvAddr("124.57.180.156:9876");
        // 启动Producer
        producer.start();

        // 创建消息对象，topic为：myTopic001，消息内容为：hello world async
        Message msg = new Message("myTopic001", "hello world async".getBytes());
        // 进行异步发送，通过SendCallback接口来得知发送的结果
        producer.send(msg, new SendCallback() {
            // 发送成功的回调接口
            @Override
            public void onSuccess(SendResult sendResult) {
                System.out.println("发送消息成功！result is : " + sendResult);
            }
            // 发送失败的回调接口
            @Override
            public void onException(Throwable throwable) {
                throwable.printStackTrace();
                System.out.println("发送消息失败！result is : " + throwable.getMessage());
            }
        });

        Thread.sleep(1000);

        producer.shutdown();
        System.out.println("生产者 shutdown！");
    }
}
```

输出结果：

```java
发送消息成功！result is : SendResult [sendStatus=SEND_OK, msgId=A9FE854106E418B4AAC26F8719B20000, offsetMsgId=7B39B49D00002A9F0000000000058CFC, messageQueue=MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=1], queueOffset=2]
生产者 shutdown！
```

### 1.4、sendOneway

```java
public class ProducerOneWay {
    public static void main(String[] args) throws Exception {
        // 指定生产组名为my-producer
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        // 配置namesrv地址
        producer.setNamesrvAddr("124.57.180.156:9876");
        // 启动Producer
        producer.start();

        // 创建消息对象，topic为：myTopic001，消息内容为：hello world oneway
        Message msg = new Message("myTopic001", "hello world oneway".getBytes());
        // 效率最高，因为oneway不关心是否发送成功，我就投递一下我就不管了。所以返回是void
        producer.sendOneway(msg);
        System.out.println("投递消息成功！，注意这里是投递成功，而不是发送消息成功哦！因为我sendOneway也不知道到底成没成功，我没返回值的。");
        producer.shutdown();
        System.out.println("生产者 shutdown！");
    }
}
```

输出结果：

```
投递消息成功！，注意这里是投递成功，而不是发送消息成功哦！因为我sendOneway也不知道到底成没成功，我没返回值的。
生产者 shutdown！
```

### 1.5、效率对比

sendOneway > sendCallBack > send批量 > send单条

很容易理解，sendOneway不求结果，我就负责投递，我不管你失败还是成功，相当于中转站，来了我就扔出去，我不进行任何其他处理。所以最快。

而sendCallBack是异步发送肯定比同步的效率高。

send批量和send单条的效率也是分情况的，如果只有1条msg要发，那还搞毛批量，直接send单条完事。

## 2、Consumer

**每个consumer只能关注一个topic。**

发消息肯定要必备如下几个条件：

- 指定消费组名（不能用默认的，会报错）
- 配置namesrv地址（必须）
- 指定topic name（必须）
- 指定tag/key（可选）

### 2.1、CLUSTERING

集群模式，默认。

比如启动五个Consumer，Producer生产一条消息后，Broker会选择五个Consumer中的其中一个进行消费这条消息，所以他属于点对点消费模式。

```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        // 指定消费组名为my-consumer
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("my-consumer");
        // 配置namesrv地址
        consumer.setNamesrvAddr("124.57.180.156:9876");
        // 订阅topic：myTopic001 下的全部消息（因为是*，*指定的是tag标签，代表全部消息，不进行任何过滤）
        consumer.subscribe("myTopic001", "*");
        // 注册监听器，进行消息消息。
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                for (MessageExt msg : msgs) {
                    String str = new String(msg.getBody());
                    // 输出消息内容
                    System.out.println(str);
                }
                // 默认情况下，这条消息只会被一个consumer消费，这叫点对点消费模式。也就是集群模式。
                // ack确认
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 启动消费者
        consumer.start();
        System.out.println("Consumer start");
    }
}
```

### 2.2、BROADCASTING

广播模式。

比如启动五个Consumer，Producer生产一条消息后，Broker会把这条消息广播到五个Consumer中，这五个Consumer分别消费一次，每个都消费一次。

```
// 代码里只需要添加如下这句话即可：
consumer.setMessageModel(MessageModel.BROADCASTING); 
```

### 2.3、两种模式对比

- 集群默认是默认的，广播模式是需要手动配置。
- 一条消息：集群模式下的多个Consumer只会有一个Consumer消费。广播模式下的每一个Consumer都会消费这条消息。
- 广播模式下，发送一条消息后，会被当前被广播的所有Consumer消费，但是后面新加入的Consumer不会消费这条消息，很好理解：村里面大喇叭喊了全村来领鸡蛋，第二天你们村新来个人，那个人肯定听不到昨天大喇叭喊的消息呀。

## 3、TAG&&KEY

> 发送/消费 消息的时候可以指定tag/key来进行过滤消息，支持通配符。*代表消费此topic下的全部消息，不进行过滤。

看下`org.apache.rocketmq.common.message.Message`源码可以发现发消息的时候可以指定tag和keys：

```java
public Message(String topic, String tags, String keys, byte[] body) {
    this(topic, tags, keys, 0, body, true);
}
```

比如：

```java
public class ProducerTagsKeys {
    public static void main(String[] args) throws Exception {
        // 指定生产组名为my-producer
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        // 配置namesrv地址
        producer.setNamesrvAddr("124.57.180.156:9876");
        // 启动Producer
        producer.start();
        // 创建消息对象，topic为：myTopic001，消息内容为：hello world，且tags为：test-tags，keys为test-keys
        Message msg = new Message("myTopic001", "test-tags", "test-keys", "hello world".getBytes());
        // 发送消息到mq，同步的
        SendResult result = producer.send(msg);
        System.out.println("发送消息成功！result is : " + result);
        // 关闭Producer
        producer.shutdown();
        System.out.println("生产者 shutdown！");
    }
}
```

输出结果：

```java
发送消息成功！result is : SendResult [sendStatus=SEND_OK, msgId=A9FE854149DC18B4AAC26FA4B7200000, offsetMsgId=7B39B49D00002A9F0000000000058DA6, messageQueue=MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=3], queueOffset=3]
生产者 shutdown！
```

查看管控台，可以发现tags和keys已经生效了：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue093pYCYJoicVLpsBmGvMY4cMwwYhe30C0syiaRjpyZdicK3mrB0rDfONQ2oMRAC3Utd0O7epl6MT3Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



消费的时候如果指定*那就是此topic下的全部消息，我们可以指定前缀通配符，比如：

```java
// 这样就只会消费myTopic001下的tag为test-*开头的消息。
consumer.subscribe("myTopic001", "test-*");

// 代表订阅Topic为myTopic001下的tag为TagA或TagB的所有消息
consumer.subscribe("myTopic001", "TagA||TagB");
```

还支持SQL表达式过滤，不是很常用。不BB了。

## 4、常见错误

### 4.1、sendDefaultImpl call timeout

#### 4.1.1、异常

```java
Exception in thread "main" org.apache.rocketmq.remoting.exception.RemotingTooMuchRequestException: sendDefaultImpl call timeout
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:666)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1342)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1288)
    at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:324)
    at com.chentongwei.mq.rocketmq.Producer.main(Producer.java:18)
```

#### 4.1.2、解决

1.如果你是云服务器，首先检查安全组是否允许9876这个端口访问，是否开启了防火墙，如果开启了的话是否将9876映射了出去。

2.修改配置文件`broker.conf`，加上：

```
brokerIP1=我用的是阿里云服务器，这里是我的公网IP
```

启动namesrv和broker的时候加上本机IP（我用的是阿里云服务器，这里是我的公网IP）：

```
./bin/mqnamesrv -n IP:9876
./bin/mqbroker -n IP:9876 -c conf/broker.conf
```

### 4.2、No route info of this topic

#### 4.2.1、异常

```java
Exception in thread "main" org.apache.rocketmq.client.exception.MQClientException: No route info of this topic: myTopic001
See http://rocketmq.apache.org/docs/faq/ for further details.
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:684)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1342)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1288)
    at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:324)
    at com.chentongwei.mq.rocketmq.Producer.main(Producer.java:18)
```

#### 4.2.2、解决

很明显发送成功了，不再是刚才的超时了，但是告诉我们没有这个topic。那不能每次都手动创建呀，所以启动broker的时候可以指定参数让broker为我们自动创建。如下

```
./bin/mqbroker -n IP:9876 -c conf/broker.conf autoCreateTopicEnable=true
```



**END**