# 一、Demo

```
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;

/**
 * Description:
 *
 * @author TongWei.Chen 2020-06-21 11:32:58
 */
public class ProducerDemo {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("my-producer");
        producer.setNamesrvAddr("124.57.180.156:9876");
        producer.start();

        Message msg = new Message("myTopic001", "hello world".getBytes());
        SendResult result = producer.send(msg);
        System.out.println("发送消息成功！result is : " + result);
    }
}
```

# 二、源码剖析

## 1、准备工作

### 1.1、new DefaultMQProducer()

```
public DefaultMQProducer(final String producerGroup) {
    this(null, producerGroup, null);
}

public DefaultMQProducer(final String namespace, final String producerGroup, RPCHook rpcHook) {
    // null
    this.namespace = namespace;
    // my-producer
    this.producerGroup = producerGroup;
    // new DefaultMQProducerImpl(this, null);
    defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
}
```

> - 给producerGroup赋值
> - new DefaultMQProducerImpl()

### 1.2、setNamesrvAddr()

```
/**
 * {@link org.apache.rocketmq.client.ClientConfig}
 */
public void setNamesrvAddr(String namesrvAddr) {
    this.namesrvAddr = namesrvAddr;
}
```

> - 给namesrv赋值

## 2、启动

### 2.1、start()

```
@Override
public void start() throws MQClientException {
    this.defaultMQProducerImpl.start();
}
```

> 再次重申：篇幅不是每一行代码都进行解析，没意义。冗余的文字你看着烦我写着也闹心，只剖析核心流程以及原理。所以删减了无用的垃圾。

#### 2.1.1、defaultMQProducerImpl.start()

```
private ServiceState serviceState = ServiceState.CREATE_JUST;

public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        // 默认为CREATE_JUST状态
        case CREATE_JUST:
            //  先默认成启动失败，等最后完全启动成功的时候再置为ServiceState.RUNNING
            this.serviceState = ServiceState.START_FAILED;
   /**
    * 检查配置，比如group有没有写，是不是默认的那个名字，长度是不是超出限制了，等等一系列验证。
    */
            this.checkConfig();
   /*
             * 单例模式，获取MQClientInstance对象，客户端实例。也就是Producer所部署的机器实例对象，负责操作的主要对象。
             */
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
   /**
    * 注册producer，其实就是往producerTable map里仍key-value
    * private final ConcurrentMap<String, MQProducerInner> producerTable = 
    * new ConcurrentHashMap<String, MQProducerInner>();
    * producerTable.putIfAbsent("my-producer", DefaultMQProducerImpl);
    */
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                                            + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                                            null);
            }
   // 将topic信息存到topicPublishInfoTable这个map里
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
   
            if (startFactory) {
                // 真正的启动核心类
                mQClientFactory.start();
            }
   // 都启动完成，没报错的话，就将状态改为运行中
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                                        + this.serviceState
                                        + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                                        null);
        default:
            break;
    }

    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

    this.timer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            try {
                // 每隔1s扫描过期的请求
                RequestFutureTable.scanExpiredRequest();
            } catch (Throwable e) {
                log.error("scan RequestFutureTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

#### 2.1.2、mQClientFactory.start()

```
public void start() throws MQClientException {
    synchronized (this) {
        // 默认为CREATE_JUST状态
        switch (this.serviceState) {
            case CREATE_JUST:
                // 先默认成启动失败，等最后完全启动成功的时候再置为ServiceState.RUNNING
                this.serviceState = ServiceState.START_FAILED;
                
                // 启动请求响应通道，核心netty
                this.mQClientAPIImpl.start();
                /**
                 * 启动各种定时任务
                 * 1.每隔2分钟去检测namesrv的变化
                 * 2.每隔30s从nameserver获取topic的路由信息有没有发生变化，或者说有没有新的topic路由信息
                 * 3.每隔30s清除下线的broker
                 * 4.每隔5s持久化所有的消费进度
                 * 5.每隔1分钟检测线程池大小是否需要调整
                 */
                this.startScheduledTask();
                // 启动拉取消息服务
                this.pullMessageService.start();
                // 启动Rebalance负载均衡服务
                this.rebalanceService.start();
                /**
                 * 这里再次调用了DefaultMQProducerImpl().start()方法，这TM不死循环了吗？
                 * 不会的，因为他传递了false，false再DefaultMQProducerImpl().start()方法里不会再次调用mQClientFactory.start();
                 * 但是这也重复执行了两次DefaultMQProducerImpl().start()方法里的其他逻辑，不知道为啥这么搞，没看懂。
                 */
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                // 都启动完成，没报错的话，就将状态改为运行中
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

### 2.2、总结

- 启动实例MQClientAPIImpl，这里封装了客户端与Broker进行通信的方法。
- 启动各种定时任务，与Broker之间的心跳等等。
- 启动消息拉取服务。
- 启动负载均衡服务。
- 启动默认的Producer服务（重复启动了，因为客户端一开始就启动了这个）。

## 3、发送消息

### 3.1、new Message

```
public Message(String topic, byte[] body) {
    this(topic, "", "", 0, body, true);
}

public Message(String topic, String tags, String keys, int flag, byte[] body, boolean waitStoreMsgOK) {
    this.topic = topic;
    this.flag = flag;
    this.body = body;
    if (tags != null && tags.length() > 0)
        this.setTags(tags);
    if (keys != null && keys.length() > 0)
        this.setKeys(keys);
    this.setWaitStoreMsgOK(waitStoreMsgOK);
}
```

> - 拼凑消息体：消息内容、tag、keys、topic等。

### 3.2、producer.send(msg)

```
public SendResult send(
    Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 参数校验
    Validators.checkMessage(msg, this);
    // 设置topic
    msg.setTopic(withNamespace(msg.getTopic()));
    // 发送消息
    return this.defaultMQProducerImpl.send(msg);
}
```

#### 3.2.1、sendDefaultImpl

```
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 检查Producer上是否是RUNNING状态
    this.makeSureStateOK();
    // 消息格式的校验
    Validators.checkMessage(msg, this.defaultMQProducer);
    // 尝试获取topic的路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        // 选择消息要发送的队列
        MessageQueue mq = null;
        // 发送结果
        SendResult sendResult = null;
        // 自动重试次数，this.defaultMQProducer.getRetryTimesWhenSendFailed()默认为2，如果是同步发送，默认重试3，否则重试1次
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        for (; times < timesTotal; times++) {
            // 选择topic的一个queue，然后往这个queue里发消息。
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                mq = mqSelected;
                try {
                    // 真正的发消息方法
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
      // 同步的，将返回的结果返回，如果返回结果状态不是成功的，则continue，进入下一次循环进行重试。    
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }
                            return sendResult;
                        default:
                            break;
                    }
                } catch (RemotingException e) {
                    continue;
                } catch (MQClientException e) {
                    continue;
                } catch (...) {...}
            } else {
                break;
            }
        }

        if (sendResult != null) {
            return sendResult;
        }
    }
}
```

## 4、RocketMQ是如何发消息的

首先需要配置好生产者组名、namesrv地址和topic以及要发送的消息内容，然后启动Producer的start()方法，启动完成后调用send()方法进行发送。

start()方法内部会进行检查namesrv、生产者组名等参数验证，然后内部会获取一个mQClientFactory对象，此对象内包含了所有与Broker进行通信的api，然后通过mQClientFactory启动请求响应通道，主要是netty，接下来启动一些定时任务，比如与broker的心跳等，还会启动负载均衡服务等，最后都启动成功的话将服务的状态标记为RUNNING。

启动完成后调用send()方法发消息，有三种发送方式，同步、异步、oneWay，都大同小异，唯一的区别的就是异步的多个线程池去异步调用发送请求，而同步则是当前请求线程直接同步调用的，核心流程都是：

先选择一个合适的queue来存储消息，选择完后拼凑一个header参数对象，通过netty的形式发送给broker。

这里值得注意的是：如果发送失败的话他会自动重试，默认同步发送的次数是3次，也就是失败后会自动重试2次。

> 怎么选择的queue，怎么做的负载均衡算法，broker收到后怎么持久化存储的等等等等后续陆续出。篇幅过长的话谁看着都脑壳痛！！！

## 5、设计模式

### 5.1、单例模式

`2.1.1、defaultMQProducerImpl.start()`这部分已经提到了，如下代码

```
/*
 * 单例模式，获取MQClientInstance对象，客户端实例。也就是Producer所部署的机器实例对象，负责操作的主要对象。
 */
this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
```

深究下看看是哪种单例：

```
public class MQClientManager {
    // 直接new
    private static MQClientManager instance = new MQClientManager();
    // 私有构造器
    private MQClientManager() {
    }
    // getInstance
    public static MQClientManager getInstance() {
        return instance;
    }
}
```

> 嗯...好吧，是个简单粗暴的饿汉式。

### 5.2、状态模式

`2.1.1、defaultMQProducerImpl.start()`还是这个，这里面大量的`switch..case`，其实这是个不规范的状态模式。先看下状态模式的定义：

*状态模式允许一个对象在其内部状态改变时改变它的行为，对象看起来就像是改变了它的类。*

再去分析上段代码，不就是使用一个成员变量 serviceState 来记录和管理自身的服务状态吗？只是与标准的状态模式不同的是它没有使用状态子类，而是使用`switch-case`来实现不同状态下的不同行为的。

### 5.3、门面模式

先来看下门面模式定义：

*门面模式主要的作用是给客户端提供了一个可以访问系统的接口，隐藏系统内部的复杂性。*

再来看RocketMQ的Producer，典型的是这种：*提供了一个可以访问系统的接口，隐藏系统内部的复杂性*的特性。

代码层面的话很明显，我们new的是`DefaultMQProducer`对象，但内部实际操作的确都是`DefaultMQProducerImpl`对象。比如源码中的start和send方法都是

```
/**
 * {@link org.apache.rocketmq.client.producer.DefaultMQProducer}
 */
public void start() throws MQClientException {
    // DefaultMQProducerImpl的start
    this.defaultMQProducerImpl.start();
}

/**
 * {@link org.apache.rocketmq.client.producer.DefaultMQProducer}
 */
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // DefaultMQProducerImpl的send
    return this.defaultMQProducerImpl.send(msg);
}
```

### 5.4、总结

吸取大牛写代码的经验，这种模式再开发需求中都能用到。学源码为了什么？

1：面试吹牛逼

2：学习优秀的代码设计思想

## 6、时序图

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucMqg1ggTT1qrlVoW3h8Snew0fuBNC1s9nbV74HAYaBSSxmzgju5PaVIaRRibV8IHDicnRAvADID7kw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**END**