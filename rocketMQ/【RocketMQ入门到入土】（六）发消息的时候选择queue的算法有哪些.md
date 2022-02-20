# 一、说明

分为两种，一种是直接发消息，client内部有选择queue的算法，不允许外界改变。还有一种是可以自定义queue的选择算法（内置了三种算法，不喜欢的话可以自定义算法实现）。

```
public class org.apache.rocketmq.client.producer.DefaultMQProducer {
    // 只发送消息，queue的选择由默认的算法来实现
    @Override
    public SendResult send(Collection<Message> msgs) {}
    
    // 自定义选择queue的算法进行发消息
    @Override
    public SendResult send(Collection<Message> msgs, MessageQueue messageQueue) {}
}
```

# 二、源码

## 1、send(msg, mq)

### 1.1、使用场景

有时候我们不希望默认的queue选择算法，而是需要自定义，一般最常用的场景在**顺序消息**，顺序消息的发送一般都会指定某组特征的消息都发当同一个queue里，这样才能保证顺序，因为单queue是有序的。

> 对顺序消息不明白的请看我之前的顺序消息文章。

### 1.2、原理剖析

内置了三种算法，三种算法都实现了一个共同的接口：

```
org.apache.rocketmq.client.producer.MessageQueueSelector
```

- `SelectMessageQueueByRandom`
- `SelectMessageQueueByHash`
- `SelectMessageQueueByMachineRoom`
- 要想自定义逻辑的话，直接实现接口重写select方法即可。

很典型的**策略模式**，不同算法不同实现类，有个顶层接口。

#### 1.2.1、SelectMessageQueueByRandom

```
public class SelectMessageQueueByRandom implements MessageQueueSelector {
    private Random random = new Random(System.currentTimeMillis());
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        // mqs.size()：队列的个数。假设队列个数是4，那么这个value就是0-3之间随机。
        int value = random.nextInt(mqs.size());
        return mqs.get(value);
    }
}
```

> so easy，就是纯随机。
>
> mqs.size()：队列的个数。假设队列个数是4，那么这个value就是0-3之间随机。

#### 1.2.2、SelectMessageQueueByHash

```
public class SelectMessageQueueByHash implements MessageQueueSelector {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = arg.hashCode();
        // 防止出现负数，取个绝对值，这也是我们平时开发中需要注意到的点
        if (value < 0) {
            value = Math.abs(value);
        }
        // 直接取余队列个数。
        value = value % mqs.size();
        return mqs.get(value);
    }
}
```

> so easy，就是纯取余。
>
> mqs.size()：队列的个数。假设队列个数是4，且value的hashcode是3，那么3 % 4 = 3，那么就是最后一个队列，也就是四号队列，因为下标从0开始。

#### 1.2.3、SelectMessageQueueByMachineRoom

```
public class SelectMessageQueueByMachineRoom implements MessageQueueSelector {
    private Set<String> consumeridcs;
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        return null;
    }
    public Set<String> getConsumeridcs() {
        return consumeridcs;
    }
    public void setConsumeridcs(Set<String> consumeridcs) {
        this.consumeridcs = consumeridcs;
    }
}
```

> 没看懂有啥鸟用，直接return null; 所以如果有自定义需求的话直接自定义就好了，这玩意没看出有啥卵用。

#### 1.2.4、自定义算法

```
public class MySelectMessageQueue implements MessageQueueSelector {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        return mqs.get(0);
    }
}
```

> 永远都选择0号队列，也就是第一个队列。只是举个例子，实际看你业务需求。

### 1.3、调用链

```
org.apache.rocketmq.client.producer.DefaultMQProducer#send(Message msg, MessageQueueSelector selector, Object arg)
->
org.apache.rocketmq.client.producer.DefaultMQProducer#send(Message msg, MessageQueueSelector selector, Object arg)
->
org.apache.rocketmq.client.producer.DefaultMQProducer#send(Message msg, MessageQueueSelector selector, Object arg, long timeout)
->
org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendSelectImpl(xxx)
->
mq = mQClientFactory.getClientConfig().queueWithNamespace(selector.select(messageQueueList, userMessage, arg));
->
selector.select(messageQueueList, userMessage, arg)
->
org.apache.rocketmq.client.producer.MessageQueueSelector#select(final List<MessageQueue> mqs, final Message msg, final Object arg)
```

## 2、send(msg)

### 2.1、使用场景

一般没特殊需求的场景都用这个。因为他默认的queue选择算法很不错，各种优化场景都替我们想到了。

### 2.2、原理剖析

```
// {@link org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl}
// 这是发送消息核心原理，不清楚的看我之前发消息源码分析的文章

// 选择消息要发送的队列
MessageQueue mq = null;
for (int times = 0; times < 3; times++) {
    // 首次肯定是null
    String lastBrokerName = null == mq ? null : mq.getBrokerName();
    // 调用下面的方法进行选择queue
    MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
    if (mqSelected != null) {
        // 给mq赋值，如果首次失败了，那么下次重试的时候（也就是下次for的时候），mq就有值了。
        mq = mqSelected;
        ......
        // 很关键，能解答下面会提到的两个问题：
        // 1.faultItemTable是什么时候放进去的？
        // 2.isAvailable() 为什么只是判断一个时间就可以知道Broker是否可用？   
        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);    
    }
}
```

选择queue的主入口

```
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    // 默认为false，代表不启用broker故障延迟
    if (this.sendLatencyFaultEnable) {
        try {
            // 随机数且+1
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            // 遍历
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                // 先（随机数 +1） % queue.size()
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0) {
                    pos = 0;
                }
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                // 看找到的这个queue所属的broker是不是可用的
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    // 非失败重试，直接返回到的队列
                    // 失败重试的情况，如果和选择的队列是上次重试是一样的，则返回
                    
                    // 也就是说如果你这个queue所在的broker可用，
                    // 且不是重试进来的或失败重试的情况，如果和选择的队列是上次重试是一样的，那你就是天选之子了。
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName)) {
                        return mq;
                    }
                }
            }
            
   // 如果所有队列都不可用，那么选择一个相对好的broker，不考虑可用性的消息队列
            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }
  // 随机选择一个queue
        return tpInfo.selectOneMessageQueue();
    }
 // 当sendLatencyFaultEnable=false的时候选择queue的方法，默认就是false。
    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

#### 2.2.1、不启用broker故障延迟

既然sendLatencyFaultEnable默认是false，那就先看当sendLatencyFaultEnable=false时候的逻辑

```
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    // 第一次就是null，第二次（也就是重试的时候）就不是null了。
    if (lastBrokerName == null) {
        // 第一次选择队列的逻辑
        return selectOneMessageQueue();
    } else {
        // 第一次选择队列发送消息失败了，第二次重试的时候选择队列的逻辑
        
        int index = this.sendWhichQueue.getAndIncrement();
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
   // 过滤掉上次发送消息失败的队列
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        return selectOneMessageQueue();
    }
}
```

那就继续看第一次选择队列的逻辑：

```
public MessageQueue selectOneMessageQueue() {
    // 当前线程有个ThreadLocal变量，存放了一个随机数 {@link org.apache.rocketmq.client.common.ThreadLocalIndex#getAndIncrement}
    // 然后取出随机数根据队列长度取模且将随机数+1
    int index = this.sendWhichQueue.getAndIncrement();
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0) {
        pos = 0;
    }
    return this.messageQueueList.get(pos);
}
```

> 好吧，其实也有点随机一个的意思。但是亮点在于取出随机数根据队列长度取模且将随机数+1，这个+1亮了（getAndIncrement cas +1）。
>
> 当消息第一次发送失败时，lastBrokerName会存放当前选择失败的broker（mq = mqSelected），通过重试，此时lastBrokerName有值，代表上次选择的boker发送失败，则重新对sendWhichQueue本地线程变量+1，遍历选择消息队列，直到不是上次的broker，也就是为了规避上次发送失败的broker的逻辑所在。
>
> 举个例子：你这次随机数是1，队列长度是4，1%4=1，这时候失败了，进入重试，那么重试之前，也就是在上一步1%4之后，他把1进行了++操作，变成了2，那么你这次重试的时候就是2%4=2，直接过滤掉了刚才失败的broker。

那就继续看第二次重试选择队列的逻辑：

```
// +1
int index = this.sendWhichQueue.getAndIncrement();
for (int i = 0; i < this.messageQueueList.size(); i++) {
    // 取模
    int pos = Math.abs(index++) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    MessageQueue mq = this.messageQueueList.get(pos);
    // 过滤掉上次发送消息失败的队列
    if (!mq.getBrokerName().equals(lastBrokerName)) {
        return mq;
    }
}
// 没找到能用的queue的话继续走默认的那个
return selectOneMessageQueue();
```

> so easy，你上次不是失败了，进入我这里重试来了吗？我也很简单，我就还是取出随机数+1然后取模队列长度，我看这个broker是不是上次失败的那个，是他小子的话就过滤掉，继续遍历queue找下一个能用的。

#### 2.2.2、启用broker故障延迟

也就是下面if里的逻辑

```
if (this.sendLatencyFaultEnable) {
    ....
}
```

看上面的注释就行了，很清晰了，就是我先`（随机数 +1） % queue.size()`，然后看你这个queue所属的broker是否可用，可用的话且不是重试进来的或失败重试的情况，如果和选择的队列是上次重试是一样的，那直接return你就完事了。那么怎么看broker是否可用的呢？

```
// {@link org.apache.rocketmq.client.latency.LatencyFaultToleranceImpl#isAvailable(String)}
public boolean isAvailable(final String name) {
    final FaultItem faultItem = this.faultItemTable.get(name);
    if (faultItem != null) {
        return faultItem.isAvailable();
    }
    return true;
}

// {@link org.apache.rocketmq.client.latency.LatencyFaultToleranceImpl.FaultItem#isAvailable()}
public boolean isAvailable() {
    return (System.currentTimeMillis() - startTimestamp) >= 0;
}
```

> 疑问：
>
> - faultItemTable是什么时候放进去的？
> - isAvailable() 为什么只是判断一个时间就可以知道Broker是否可用？

这就需要上面发送消息完成后所调用的这个方法了：

```
// {@link org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#updateFaultItem}
// 发送开始时间
beginTimestampPrev = System.currentTimeMillis();
// 进行发送
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
// 发送结束时间
endTimestamp = System.currentTimeMillis();
// 更新broker的延迟情况
this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
```

细节逻辑如下：

```
// {@link org.apache.rocketmq.client.latency.MQFaultStrategy#updateFaultItem}
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
    if (this.sendLatencyFaultEnable) {
        // 首次isolation传入的是false，currentLatency是发送消息所耗费的时间，如下
        // this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
        long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
        this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
    }
}

private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};

// 根据延迟时间对比MQFaultStrategy中的延迟级别数组latencyMax 不可用时长数组notAvailableDuration 来将该broker加进faultItemTable中。
private long computeNotAvailableDuration(final long currentLatency) {
    for (int i = latencyMax.length - 1; i >= 0; i--) {
        // 假设currentLatency花费了10ms，那么latencyMax里的数据显然不符合下面的所有判断，所以直接return 0;
        if (currentLatency >= latencyMax[i])
            return this.notAvailableDuration[i];
    }
    return 0;
}

// {@link org.apache.rocketmq.client.latency.LatencyFaultToleranceImpl#updateFaultItem()}
@Override
// 其实主要就是给startTimestamp赋值为当前时间+computeNotAvailableDuration(isolation ? 30000 : currentLatency);的结果，给isAvailable()所用
// 也就是说只有notAvailableDuration == 0的时候，isAvailable()才会返回true。
public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
    FaultItem old = this.faultItemTable.get(name);
    if (null == old) {
        final FaultItem faultItem = new FaultItem(name);
        faultItem.setCurrentLatency(currentLatency);
        // 给startTimestamp赋值为当前时间+computeNotAvailableDuration(isolation ? 30000 : currentLatency);的结果，给isAvailable()所用
        faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

        old = this.faultItemTable.putIfAbsent(name, faultItem);
        if (old != null) {
            old.setCurrentLatency(currentLatency);
            // 给startTimestamp赋值为当前时间+computeNotAvailableDuration(isolation ? 30000 : currentLatency);的结果，给isAvailable()所用
            old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
        }
    } else {
        old.setCurrentLatency(currentLatency);
        // 给startTimestamp赋值为当前时间+computeNotAvailableDuration(isolation ? 30000 : currentLatency);的结果，给isAvailable()所用
        old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
    }
}
```

下面这两句代码详细解释下：

```
private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
```

| latencyMax | notAvailableDuration |
| :--------: | :------------------: |
|    50L     |          0L          |
|    100L    |          0L          |
|    550L    |        30000L        |
|   1000L    |        60000L        |
|   2000L    |       120000L        |
|   3000L    |       180000L        |
|   15000L   |       600000L        |

即

- currentLatency大于等于50小于100，则notAvailableDuration为0
- currentLatency大于等于100小于550，则notAvailableDuration为0
- currentLatency大于等于550小于1000，则notAvailableDuration为300000
- …等等

再来举个例子：

假设isolation传入true，

```
long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
```

那么notAvailableDuration将传入600000L。结合isAvailable方法，大概流程如下：

RocketMQ为每个Broker预测了个可用时间(当前时间+notAvailableDuration)，当当前时间大于该时间，才代表Broker可用，而notAvailableDuration有6个级别和latencyMax的区间一一对应，根据传入的currentLatency去预测该Broker在什么时候可用。

所以再来看这个

```
public boolean isAvailable() {
    return (System.currentTimeMillis() - startTimestamp) >= 0;
}
```

根据执行时间来看落入哪个区间，在0~100的时间内notAvailableDuration都是0，都是可用的，大于该值后，可用的时间就会开始变大了，就认为不是最优解，直接舍弃。

### 2.3、调用链

```
org.apache.rocketmq.client.producer.DefaultMQProducer#send(org.apache.rocketmq.common.message.Message)
->
org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#send(org.apache.rocketmq.common.message.Message)
->
org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#send(org.apache.rocketmq.common.message.Message, long)
->
org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl(xxx)
->
MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
->
org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#selectOneMessageQueue(xxx) 
org.apache.rocketmq.client.latency.MQFaultStrategy#selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName)    
```

### 2.4、总结

- 在不开启容错的情况下，轮询队列进行发送，如果失败了，重试的时候过滤失败的Broker
- 如果开启了容错策略，会通过RocketMQ的预测机制来预测一个Broker是否可用
- 如果上次失败的Broker可用那么还是会选择该Broker的队列
- 如果上述情况失败，则随机选择一个进行发送
- 在发送消息的时候会记录一下调用的时间与是否报错，根据该时间去预测broker的可用时间

# 三、总结

## 1、疑问

他搞了两个重载send()方法，一个支持算法选择器，一个不支持算法选择，queue的算法选择是个典型的策略模式。为什么`send(message)`方法内置的queue选择算法不抽出到单独的类中，然后此类实现`org.apache.rocketmq.client.producer.MessageQueueSelector`接口呢？比如叫：`SelectMessageQueueByBest`，比如如下：

```
public class org.apache.rocketmq.client.producer.DefaultMQProducer {
    // 只发送消息，queue的选择由默认的算法来实现
    @Override
    public SendResult send(Collection<Message> msgs) {
        this.send(msgs, new SelectMessageQueueByBest().select(xxx));
    }
    
    // 自定义选择queue的算法进行发消息
    @Override
    public SendResult send(Collection<Message> msgs, MessageQueue messageQueue) {}
}
```

我猜测可能是这个算法过于复杂，与其它类的交互也过于多，参数也可能和内置的其他三个不同，所以没搞到一起，但是还是搞到一起规范呀，干的同一件事，只是算法不同，很典型的策略模式。

## 2、发消息的时候选择queue的算法有哪些

分为两种，一种是直接发消息，不能选择queue，这种的queue选择算法如下：

- 在不开启容错的情况下，轮询队列进行发送，如果失败了，重试的时候过滤失败的Broker
- 如果开启了容错策略，会通过RocketMQ的预测机制来预测一个Broker是否可用
- 如果上次失败的Broker可用那么还是会选择该Broker的队列
- 如果上述情况失败，则随机选择一个进行发送
- 在发送消息的时候会记录一下调用的时间与是否报错，根据该时间去预测broker的可用时间

另外一种是发消息的时候可以选择算法甚至还可以实现接口自定义算法：

- `SelectMessageQueueByRandom`：随机
- `SelectMessageQueueByHash`：hash
- `SelectMessageQueueByMachineRoom`
- 实现接口自定义

**END**