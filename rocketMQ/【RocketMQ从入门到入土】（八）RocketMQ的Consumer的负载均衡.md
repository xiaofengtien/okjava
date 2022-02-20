# 一、问题描述

RocketMQ的Consumer是如何做的负载均衡？比如：5个Consumer进程同时消费一个Topic，这个Topic只有4个queue会出现啥情况？反之Consumer数量小于queue的数据是啥情况？

# 二、源码剖析

## 1、RebalancePushImpl

```
public class RebalancePushImpl extends RebalanceImpl {
    public RebalancePushImpl(String consumerGroup, MessageModel messageModel,
        AllocateMessageQueueStrategy allocateMessageQueueStrategy,
        MQClientInstance mQClientFactory, DefaultMQPushConsumerImpl defaultMQPushConsumerImpl) {
        // 可以看到很简单，调用了父类RebalanceImpl的构造器
        super(consumerGroup, messageModel, allocateMessageQueueStrategy, mQClientFactory);
        this.defaultMQPushConsumerImpl = defaultMQPushConsumerImpl;
    }
```

## 2、RebalanceImpl

```
public abstract class RebalanceImpl {
    // 很简单，就是初始化一些东西，关键在于下面的doRebalance
    public RebalanceImpl(String consumerGroup, MessageModel messageModel,
        AllocateMessageQueueStrategy allocateMessageQueueStrategy,
        MQClientInstance mQClientFactory) {
        this.consumerGroup = consumerGroup;
        this.messageModel = messageModel;
        this.allocateMessageQueueStrategy = allocateMessageQueueStrategy;
        this.mQClientFactory = mQClientFactory;
    }
    
    /**
     * 分配消息队列，命名抄袭spring，doXXX开始真正的业务逻辑
     *
     * @param isOrder：是否是顺序消息 true：是；false：不是
     */
    public void doRebalance(final boolean isOrder) {
        // 分配每个topic的消息队列
        Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
        if (subTable != null) {
            for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
                final String topic = entry.getKey();
                try {
                    // 这个是关键了
                    this.rebalanceByTopic(topic, isOrder);
                } catch (Throwable e) {
                    if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                        log.warn("rebalanceByTopic Exception", e);
                    }
                }
            }
        }

        // 移除未订阅的topic对应的消息队列
        this.truncateMessageQueueNotMyTopic();
    }
}
```

### 2.1、rebalanceByTopic

```
private void rebalanceByTopic(final String topic, final boolean isOrder) {
    switch (messageModel) {
        case CLUSTERING: {
            // 获取topic对应的队列和consumer信息，比如mqSet如下
            /**
             * 0 = {MessageQueue@2151} "MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=3]"
             * 1 = {MessageQueue@2152} "MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=0]"
             * 2 = {MessageQueue@2153} "MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=2]"
             * 3 = {MessageQueue@2154} "MessageQueue [topic=myTopic001, brokerName=broker-a, queueId=1]"
             */
            Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
            // 所有的Consumer客户端cid，比如：172.16.20.246@7832
            List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
            if (mqSet != null && cidAll != null) {
                List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
                // 为什么要addAll到list里，因为他要排序
                mqAll.addAll(mqSet);

                // 排序消息队列和消费者数组，因为是在进行分配队列，排序后，各Client的顺序才能保持一致。
                Collections.sort(mqAll);
                Collections.sort(cidAll);

                // 默认选择的是org.apache.rocketmq.client.consumer.rebalance.AllocateMessageQueueAveragely
                AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

                // 根据队列分配策略分配消息队列
                List<MessageQueue> allocateResult = null;
                try {
                    // 这个才是要介绍的真正C位，strategy.allocate()
                    allocateResult = strategy.allocate(
                        this.consumerGroup,
                        this.mQClientFactory.getClientId(),
                        mqAll,
                        cidAll);
                } catch (Throwable e) {
                    return;
                }
            }
        }
    }
}
```

## 3、AllocateMessageQueueAveragely

### 3.1、allocate

```
public class AllocateMessageQueueAveragely implements AllocateMessageQueueStrategy {
    private final InternalLogger log = ClientLogger.getLog();
    @Override
    public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
        List<String> cidAll) {
        /**
         * 参数校验的代码我删了。
         */
        
        List<MessageQueue> result = new ArrayList<MessageQueue>();
        /**
         * 第几个Consumer，这也是我们上面为什么要排序的重要原因之一。
         * Collections.sort(mqAll);
         * Collections.sort(cidAll);
         */
        int index = cidAll.indexOf(currentCID);
        // 取模，多少消息队列无法平均分配 比如mqAll.size()是4，代表4个queue。cidAll.size()是5，代表一个consumer，那么mod就是4
        int mod = mqAll.size() % cidAll.size();
        // 平均分配
        // 4 <= 5 ? 1 : (4 > 0 && 1 < 4 ? 4 / 5 + 1 : 4 / 5)
        int averageSize =
            mqAll.size() <= cidAll.size() ? 1 : (mod > 0 && index < mod ? mqAll.size() / cidAll.size()
                + 1 : mqAll.size() / cidAll.size());
        // 有余数的情况下，[0, mod) 平分余数，即每consumer多分配一个节点；第index开始，跳过前mod余数。
        int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod;
        // 分配队列数量。之所以要Math.min()的原因是，mqAll.size() <= cidAll.size()，部分consumer分配不到消息队列。
        int range = Math.min(averageSize, mqAll.size() - startIndex);
        for (int i = 0; i < range; i++) {
            result.add(mqAll.get((startIndex + i) % mqAll.size()));
        }
        return result;
    }
}
```

### 3.2、解释

看着这算法凌乱的很，太复杂了！说实话，确实挺复杂，蛮罗嗦的，但是代数法可以得到如下表格：

| 假设4个queue | Consumer有2个 *可以整除* | Consumer有3个 *不可整除* | Consumer有5个 *无法都分配* |
| :----------- | :----------------------- | :----------------------- | :------------------------- |
| queue[0]     | Consumer[0]              | Consumer[0]              | Consumer[0]                |
| queue[1]     | Consumer[0]              | Consumer[0]              | Consumer[1]                |
| queue[2]     | Consumer[1]              | Consumer[1]              | Consumer[2]                |
| queue[3]     | Consumer[1]              | Consumer[2]              | Consumer[3]                |

所以得出如下真香定律（也是回击面试官的最佳答案）：

- queue个数大于Consumer个数，且queue个数能整除Consumer个数的话， 那么Consumer会平均分配queue。（比如上面表格的**Consumer有2个 可以整除**部分）
- queue个数大于Consumer个数，且queue个数不能整除Consumer个数的话， 那么会有一个Consumer多消费1个queue，其余Consumer平均分配。（比如上面表格的**Consumer有3个 不可整除**部分）
- queue个数小于Consumer个数，那么会有Consumer闲置，就是浪费掉了，其余Consumer平均分配到queue上。（比如上面表格的**Consumer有5个 无法都分配**部分）

## 4、补充

queue选择算法也就是负载均衡算法有很多种可选择：

- `AllocateMessageQueueAveragely`：是前面讲的默认方式
- `AllocateMessageQueueAveragelyByCircle`：每个消费者依次消费一个partition，环状。
- `AllocateMessageQueueConsistentHash`：一致性hash算法
- `AllocateMachineRoomNearby`：就近元则，离的近的消费
- `AllocateMessageQueueByConfig`：是通过配置的方式

# 三、何时Rebalance

那就得从Consumer启动的源码开始看起，先看Consumer的启动方法start()

```
public class DefaultMQPushConsumerImpl implements MQConsumerInner {
    private MQClientInstance mQClientFactory;
    
    // 启动Consumer的入口函数
 public synchronized void start() throws MQClientException {
        this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(
            this.defaultMQPushConsumer, this.rpcHook);
        // 调用MQClientInstance的start方法，追进去看看。
        mQClientFactory.start();
    }
}
```

看看`mQClientFactory.start();`都干了什么

```
public class MQClientInstance {
    private final RebalanceService rebalanceService;
    
    public void start() throws MQClientException {
        synchronized (this) {
            // 调用RebalanceService的start方法，别慌，继续追进去看看
   this.rebalanceService.start();
        }
    }
}
```

看看`rebalanceService.start();`都干了什么，先看下他的父类`ServiceThread`

```
/*
 * 首先可以发现他是个线程的任务，实现了Runnable接口
 * 其次发现上步调用的start方法居然就是thread.start()，那就相当于调用了RebalanceService的run方法
 */
public abstract class ServiceThread implements Runnable {
 public void start() {
        this.thread = new Thread(this, getServiceName());
        this.thread.setDaemon(isDaemon);
        this.thread.start();
    }
}
```

最后来看看`RebalanceService.run()`

```
public class RebalanceService extends ServiceThread {
    /**
     * 等待时间的间隔，毫秒，默认是20s
     */
    private static long waitInterval =
        Long.parseLong(System.getProperty(
            "rocketmq.client.rebalance.waitInterval", "20000"));

    @Override
    public void run() {
        while (!this.isStopped()) {
            // 等待20s，然后超时自动释放锁执行doRebalance
            this.waitForRunning(waitInterval);
            this.mqClientFactory.doRebalance();
        }
    }
}
```

到这里真相大白了。

当一个consumer出现宕机后，默认最多20s，其它机器将重新消费已宕机的机器消费的queue，同样当有新的Consumer连接上后，20s内也会完成rebalance使得新的Consumer有机会消费queue里的msg。

等等，好像有问题：新上线一个Consumer要等20s才能负载均衡？这不是搞笑呢吗？肯定有猫腻。

确实，新启动Consumer的话会立即唤醒沉睡的线程， 让他立马进行`this.mqClientFactory.doRebalance();`，源码如下

```
public class DefaultMQPushConsumerImpl implements MQConsumerInner {
    // 启动Consumer的入口函数
 public synchronized void start() throws MQClientException {        
        // 看到了没！！！， 见名知意，立即rebalance负载均衡
        this.mQClientFactory.rebalanceImmediately();
    }
}
```



**END**