# 一、为什么要集群

- 单点存在单点故障问题
- 集群可以分担压力，提高QPS
- 主从可以保证消息可靠性，比如只有M没S。M磁盘坏了，那未被消费的消息都丢了。而S可以作为备份。

# 二、单M模式

## 1、特点

- 只有一个Master节点，所以单点故障是致命缺点。
- 优点：配置简单，方便部署。
- 缺点：单点故障，一旦Broker重启或者直接宕机了，那会导致整个服务不可用。

## 2、图解

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

# 三、多M模式

## 1、特点

- 一个集群无Slave节点，全是Master节点。
- 优点：配置相对不复杂，单个M宕机或者重启对业务系统无感知，照常提供服务。只是这个broker上如果有消息未被消费的话可能无法继续消费，但是消息不会丢失，持久化到磁盘的。异步刷盘的话会存在少量丢失。
- 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。

## 2、图解

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

# 四、多M多S模式

一个集群既有Master节点又有Slave节点。

## 1、异步复制

- 每个 Master 配置一个 Slave，有多对Master-Slave， HA，采用异步/同步复制方式，主备有短暂消息延迟，毫秒级。
- 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为Master 宕机后，消费者仍然可以从 Slave消费，此过程对应用透明。不需要人工干预。性能同多 Master 模式几乎一样。
- 缺点：Master 宕机，磁盘损坏情况，会丢失少量消息。

## 2、同步双写

- 每个 Master 配置一个 Slave，有多对Master-Slave， HA采用同步双写方式，主备都写成功，向应用返回成功。
- 优点：数据与服务都无单点， Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高。
- 缺点：性能比异步复制模式略低，大约低 10%左右，发送单个消息的 RT会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。
- 要想真正意义的保证消息不丢失，这个同步双写是必须的 。

## 3、图解

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucnibnkqaAiayPL8vrTXbcKib9A2npkBB6ymelyoIArzYYCe7pYg4jCwMHMD4swag8fyBtk5VMnbm2Pg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 五、queue的分布

一个topic的queue可以分布到多个Broker上。比如一个topic有4个queue，他可能分配到broker-a上三个queue，broker-b上1个queue，这个queue的分配是由broker端决定的。但是为了验证猜想我们可以手动从管控台去创建这个topic，成功的话可以验证我们的猜想。

*首先我有2M2S的一个集群*

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

*创建topic*

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

*创建topic*

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

*查看status，可以发现为我们在每个broker上都创建了4个queue，也就是一共8个queue了。*

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucnibnkqaAiayPL8vrTXbcKib923HnSFg0B9JNILCENkY10lVNX5xCNPN7TsuaD3IGD0svQtcHonzY0g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

*点击【TOPIC CONFIG】更改配置*

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucnibnkqaAiayPL8vrTXbcKib94oicmpNicd3l60yGk6BneIk862sPVIWKlOzgktBvibzsV4BKUuUEOyVKA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

*再次查看就会发现已经生效了，验证了我们的猜想*

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucnibnkqaAiayPL8vrTXbcKib90Z9icqicI6Gy5UBB6dX4HPjw4cmKiaOicPQyZydPHrAu9uibXwFpVsvaGKA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**每个queue的消息都是不一样的，也就是比如你发N条消息，他可能一部分在broker-a上一部分在broker-b上，不管他在哪，消息都是不一样的，不要理解成M-S那种复制。他只是负载均衡将queue分配到了不同的broker上。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucnibnkqaAiayPL8vrTXbcKib9icQSyfjBFw5UYsBGl3F34XoOibGaR6P9QrsKMILtKibVq8FhWU0KyB1fQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**END**