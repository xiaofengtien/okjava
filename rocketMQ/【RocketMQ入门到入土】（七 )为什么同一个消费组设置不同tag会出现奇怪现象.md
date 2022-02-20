# 一、问题复现

## 1、描述

两个一样的Consumer Group的Consumer订阅同一个Topic，但是是不同的tag，Consumer1订阅Topic的tag1，Consumer2订阅Topic的tag2，然后分别启动。这时候往Topic的tag1里发送10条数据，Topic的tag2里发送10条。目测应该是Consumer1和Consumer2分别收到对应的10条消息。结果却是只有Consumer2收到了消息，而且只收到了4-6条消息，不固定。

## 2、代码

### 2.1、Consumer

```
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("test-consumer");
        consumer.setNamesrvAddr("124.57.180.156:9876");
        consumer.subscribe("TopicTest2","tag1");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(
                    List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                MessageExt msg = msgs.get(0);
                System.out.println(msg.getTags());
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("ConsumerStarted.");
    }
}
```

> 启动这个订阅了TopicTest2的tag1标签的Consumer，然后将tag1改为tag2再次启动Consumer。这就相当于启动了两个Consumer进程，一个订阅了TopicTest2的tag1标签，另一个订阅了TopicTest2的tag2标签。

### 2.2、Producer

```
public class Producer {
    public static void main(String[] args) throws MQClientException {
        final DefaultMQProducer producer = new DefaultMQProducer("test-producer");
        producer.setNamesrvAddr("124.57.180.156:9876");
        producer.start();

        for (int i = 0; i < 10; i++){
            try {
                Message msg = new Message("TopicTest2", "tag1", ("Hello tag1 - "+i).getBytes());
                SendResult sendResult = producer.send(msg);
                System.out.println(sendResult);
            }catch(Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

> 启动Producer，往TopicTest2的tag1里发10条消息。再次将tag1改为tag2，然后再次启动Producer进行发送，这样就是TopicTest2的tag1下有10条消息，TopicTest2的tag2下也有10条消息。

## 3、结果

Consumer和Producer都启动后，发现如下：

- Producer发送了20条消息正常。
- Consumer1没有消费到tag1下的数据
- Consumer2消费了一半（不一定是几条，有时候5条，有时候6条的）消息。

# 二、问题答案

- 首先这是Broker决定的，而不是Consumer端决定的

> 之前看过一篇文章写的有理有据，写的是Consumer端，还贴出了debug的源码，说后者覆盖了前者，但是我想说：你启动了两个独立的Consumer，那是两个独立的进程，根本不存在覆盖不覆盖的问题，那就是独立的。JVM就一个。又不是共享的JVM，何来覆盖？

- Consumer端发心跳给Broker，Broker收到后存到consumerTable里（就是个Map），key是GroupName，value是ConsumerGroupInfo。
- ConsumerGroupInfo里面是包含topic等信息的，但是问题就出在上一步骤，key是groupName，你同GroupName的话Broker心跳最后收到的Consumer会覆盖前者的。相当于如下代码：

```
map.put(groupName, ConsumerGroupInfo);
```

这样同key，肯定产生了覆盖。所以Consumer1不会收到任何消息，但是Consumer2为什么只收到了一半（不固定）消息呢？

那是因为：你是集群模式消费，它会负载均衡分配到各个节点去消费，所以一半消息（不固定个数）跑到了Consumer1上，结果Consumer1订阅的是tag1，所以不会任何输出。

**如果换成BROADCASTING，那绝逼后者会收到全部消息，而不是一半，因为广播是广播全部Consumer。**

# 三、源码验证

## 1、调用链

```
# 核心在于如下这个方法
org.apache.rocketmq.broker.client.ConsumerManager#registerConsumer()
    
# 关键调用链如下
# 入口是Broker启动的时候
org.apache.rocketmq.broker.BrokerStartup#start()
org.apache.rocketmq.broker.BrokerController#start()
org.apache.rocketmq.remoting.netty.NettyRemotingServer#start() 
org.apache.rocketmq.remoting.netty.NettyRemotingServer#prepareSharableHandlers()
org.apache.rocketmq.remoting.netty.NettyRemotingServer.NettyServerHandler#channelRead0()
org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processMessageReceived()
org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processRequestCommand()
org.apache.rocketmq.broker.processor.ClientManageProcessor#processRequest()
org.apache.rocketmq.broker.processor.ClientManageProcessor#heartBeat()
org.apache.rocketmq.broker.client.ConsumerManager#registerConsumer()
```

## 2、源码

### 2.1、registerConsumer

```
/**
 * Consumer信息
 */
public class ConsumerGroupInfo {
    // 组名
    private final String groupName;
    // topic信息，比如topic、tag等
    private final ConcurrentMap<String/* Topic */, SubscriptionData> subscriptionTable =
        new ConcurrentHashMap<String, SubscriptionData>();
    // 客户端信息，比如clientId等
    private final ConcurrentMap<Channel, ClientChannelInfo> channelInfoTable =
        new ConcurrentHashMap<Channel, ClientChannelInfo>(16);
    // PULL/PUSH
    private volatile ConsumeType consumeType;
    // 消费模式：BROADCASTING/CLUSTERING
    private volatile MessageModel messageModel;
    // 消费到哪了
    private volatile ConsumeFromWhere consumeFromWhere;
}

/**
 * 通过心跳将Consumer信息注册到Broker端。
 */
public boolean registerConsumer(final String group, final ClientChannelInfo clientChannelInfo,
        ConsumeType consumeType, MessageModel messageModel, ConsumeFromWhere consumeFromWhere,
        final Set<SubscriptionData> subList, boolean isNotifyConsumerIdsChangedEnable) {

    // consumerTable：维护所有的Consumer
    ConsumerGroupInfo consumerGroupInfo = this.consumerTable.get(group);
    // 如果没有Consumer，则put到map里
    if (null == consumerGroupInfo) {
        ConsumerGroupInfo tmp = new ConsumerGroupInfo(group, consumeType, messageModel, consumeFromWhere);
        // put到map里
        ConsumerGroupInfo prev = this.consumerTable.putIfAbsent(group, tmp);
        consumerGroupInfo = prev != null ? prev : tmp;
    }

    // 更新Consumer信息，客户端信息
    boolean r1 =
        consumerGroupInfo.updateChannel(clientChannelInfo, consumeType, messageModel,
                                        consumeFromWhere);
    // 更新订阅Topic信息
    boolean r2 = consumerGroupInfo.updateSubscription(subList);

    if (r1 || r2) {
        if (isNotifyConsumerIdsChangedEnable) {
            this.consumerIdsChangeListener.handle(ConsumerGroupEvent.CHANGE, group, consumerGroupInfo.getAllChannel());
        }
    }

    this.consumerIdsChangeListener.handle(ConsumerGroupEvent.REGISTER, group, subList);

    return r1 || r2;
}
```

> 从这一步可以看出消费者信息是以groupName为key，ConsumerGroupInfo为value存到map（consumerTable）里的，那很明显了，后者肯定会覆盖前者的，因为key是一样的。而后者的tag是tag2，那肯定覆盖了前者的tag1，这部分是存到ConsumerGroupInfo的subscriptionTable里面的
>
> ```
> private final ConcurrentMap<String/* Topic */, SubscriptionData> subscriptionTable =
>     new ConcurrentHashMap<String, SubscriptionData>();
> ```
>
> SubscriptionData包含了topic等信息
>
> ```
> public class SubscriptionData implements Comparable<SubscriptionData> {
>     // topic
>     private String topic;
>     private String subString;
>     // tags
>     private Set<String> tagsSet = new HashSet<String>();
>     private Set<Integer> codeSet = new HashSet<Integer>();
> }
> ```

### 2.2、两个问题

**1.topic、tag等信息是怎么覆盖的？**

```
boolean r1 = consumerGroupInfo.updateChannel(clientChannelInfo, consumeType, messageModel,consumeFromWhere);
/**
 * 其实很简单，就是以topic为key，SubscriptionData为value。而SubscriptionData里包含了tags信息，所以直接覆盖掉
 */
public boolean updateSubscription(final Set<SubscriptionData> subList) {
    for (SubscriptionData sub : subList) {
        SubscriptionData old = this.subscriptionTable.get(sub.getTopic());
        if (old == null) {
            SubscriptionData prev = this.subscriptionTable.putIfAbsent(sub.getTopic(), sub);
        } else if (sub.getSubVersion() > old.getSubVersion()) {
            this.subscriptionTable.put(sub.getTopic(), sub);
        }
    }
}
```

等等，这里好像有新发现`ConsumerGroupInfo#subscriptionTable`

```
// {@link org.apache.rocketmq.broker.client.ConsumerGroupInfo#subscriptionTable}
private final ConcurrentMap<String/* Topic */, SubscriptionData> subscriptionTable =
        new ConcurrentHashMap<String, SubscriptionData>();
```

可以有意外收获就是topic作为map的key，那岂不是一个Consumer可以订阅多个Topic？是的，通过这段源码可以发现是没毛病的，我也测试过。

**2.这么看的话Consumer端只会存在一个进程，因为同组，注册进去就覆盖了呀？**

大哥，注意ConsumerGroupInfo里的channelInfoTable

```
// 客户端信息，比如clientId等
private final ConcurrentMap<Channel, ClientChannelInfo> channelInfoTable =
    new ConcurrentHashMap<Channel, ClientChannelInfo>(16);
```

ClientChannelInfo是包含clientId等信息的，代表一个Consumer。注册方法是：

```
boolean r2 = consumerGroupInfo.updateSubscription(subList);
/**
 * 下面是删减后的代码，其实就是以Channel作为key，每个Consumer的Channel是不一样的。所以能存多个Consumer客户端
 */
public boolean updateChannel(final ClientChannelInfo infoNew, ConsumeType consumeType,
        MessageModel messageModel, ConsumeFromWhere consumeFromWhere) {
    ClientChannelInfo infoOld = this.channelInfoTable.get(infoNew.getChannel());
    if (null == infoOld) {
        ClientChannelInfo prev = this.channelInfoTable.put(infoNew.getChannel(), infoNew);
    }
}
```

**
**

**END**