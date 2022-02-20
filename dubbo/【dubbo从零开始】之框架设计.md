# ** Dubbo框架设计**

## RPC基础

### 何为 RPC?

**RPC（Remote Procedure Call）** 即远程过程调用，通过名字我们就能看出 RPC 关注的是远程调用而非本地调用。

**为什么要 RPC  ？** 因为，两个不同的服务器上的服务提供的方法不在一个内存空间，所以，需要通过网络编程才能传递方法调用所需要的参数。并且，方法调用的结果也需要通过网络编程来接收。但是，如果我们自己手动网络编程来实现这个调用过程的话工作量是非常大的，因为，我们需要考虑底层传输方式（TCP还是UDP）、序列化方式等等方面。


**RPC 能帮助我们做什么呢？ ** 简单来说，通过 RPC 可以帮助我们调用远程计算机上某个服务的方法，这个过程就像调用本地方法一样简单。并且！我们不需要了解底层网络编程的具体细节。


举个例子：两个不同的服务 A、B 部署在两台不同的机器上，服务 A 如果想要调用服务 B 中的某个方法的话就可以通过 RPC 来做。

一言蔽之：**RPC 的出现就是为了让你调用远程方法像调用本地方法一样简单。**

### RPC 的原理是什么?

为了能够帮助小伙伴们理解 RPC 原理，我们可以将整个 RPC的 核心功能看作是下面👇 6 个部分实现的：


1. **客户端（服务消费端）** ：调用远程方法的一端。
1. **客户端 Stub（桩）** ： 这其实就是一代理类。代理类主要做的事情很简单，就是把你调用方法、类、方法参数等信息传递到服务端。
1. **网络传输** ： 网络传输就是你要把你调用的方法的信息比如说参数啊这些东西传输到服务端，然后服务端执行完之后再把返回结果通过网络传输给你传输回来。网络传输的实现方式有很多种比如最近基本的 Socket或者性能以及封装更加优秀的 Netty（推荐）。
1. **服务端 Stub（桩）** ：这个桩就不是代理类了。我觉得理解为桩实际不太好，大家注意一下就好。这里的服务端 Stub 实际指的就是接收到客户端执行方法的请求后，去指定对应的方法然后返回结果给客户端的类。
1. **服务端（服务提供端）** ：提供远程方法的一端。

具体原理图如下，后面我会串起来将整个RPC的过程给大家说一下。


![RPC原理图](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/18-12-6/37345851.jpg)

1. 服务消费端（client）以本地调用的方式调用远程服务；
1. 客户端 Stub（client stub） 接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体（序列化）：`RpcRequest`；
1. 客户端 Stub（client stub） 找到远程服务的地址，并将消息发送到服务提供端；
1. 服务端 Stub（桩）收到消息将消息反序列化为Java对象: `RpcRequest`；
1. 服务端 Stub（桩）根据`RpcRequest`中的类、方法、方法参数等信息调用本地的方法；
1. 服务端 Stub（桩）得到方法执行结果并将组装成能够进行网络传输的消息体：`RpcResponse`（序列化）发送至消费方；
1. 客户端 Stub（client stub）接收到消息并将消息反序列化为Java对象:`RpcResponse` ，这样也就得到了最终结果。

# dubbo框架设计

官方文档：http://dubbo.apache.org/zh-cn/docs/dev/design.html

## 1.1  Business部分

在Business部分，只有一个层面：Service

对于Service层，只是提供一个ServiceInterface，再在Provider中写对应的ServiceImpl即可

也就是说，对于应用而言，到这里就足以了

下面的部分全部都是Dubbo的底层原理

## **1.2  RPC部分**

自上往下，依次有：

- config 配置层：对外配置接口
  - 它是封装配置文件中的配置信息
  - 以ServiceConfig, ReferenceConfig为中心
  - 可以直接初始化配置类，也可以通过Spring解析配置生成配置类
- proxy 服务代理层：服务接口透明代理，生成服务的客户端Stub和服务器端Skeleton
  - 它实际上就是辅助生成RPC的代理对象
  - 以ServiceProx为中心，扩展接口为ProxyFactory
- registry 注册中心层：封装服务地址的注册与发现
  - 这一层就是注册中心的核心功能层(服务发现、服务注册)
  - 以服务URL为中心，扩展接口为RegistryFactory, Registry, RegistryService
- cluster 路由层：封装多个提供者的路由及负载均衡，并桥接注册中心
  - 这一层的调用者Invoker可以保证多台服务器的服务调用，以及实现负载均衡
  - 以Invoker为中心，扩展接口为Cluster, Directory, Router, LoadBalance
- monitor 监控层：RPC调用次数和调用时间监控
  - 以Statistics为中心，扩展接口为MonitorFactory, Monitor, MonitorService
- protocol 远程调用层：封装RPC调用
  - 这一层是RPC的调用核心
  - 以Invocation, Result为中心，扩展接口为Protocol, Invoker, Exporter

## **1.3  Remoting部分**

自上往下，依次有：

- exchange 信息交换层：封装请求响应模式，同步转异步
  - 这一层负责给服务提供方与服务消费方之间架起连接的管道
  - 以Request, Response为中心，扩展接口为Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer
- transport 网络传输层：抽象mina和netty为统一接口
  - 这一层负责真正的网络数据传输，Netty也就在这一层被封装
  - 以Message为中心，扩展接口为Channel, Transporter, Client, Server, Codec
- serialize 数据序列化层：可复用的一些工具
  - 扩展接口为Serialization, ObjectInput, ObjectOutput, ThreadPool

# dubbo协议

Dubbo框架默认支持的阿里的dubbo协议，同时还支持包括rmi、hessian、http、webservice、thrift、redis在内的多种协议，下面我们来了解一下这些协议的区别与适用场景。
![图片描述](https://img1.sycdn.imooc.com/5b99d48d00011f0b15420193.png)

如果觉得上面的表单过于难理解，可以忽略 O(∩_∩)O哈哈~，我们直接来点干的。
**Dubbo协议特点：** 传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用dubbo协议传输大文件或超大字符串，基于以上描述，我们一般建议Dubbo用于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。
**RMI协议特点：** 传入传出参数数据包大小混合，消费者与提供者个数差不多，可传文件。基于以上描述，我们一般对传输管道和效率没有那么高的要求，同时又有传输文件这一类的要求时，可以尝试采用RMI协议。
**Hessian协议特点：** 传入传出参数数据包大小混合，提供者比消费者个数多，可用浏览器查看，可用表单或URL传入参数，暂不支持传文件。比较适用于需同时给应用程序和浏览器JS使用的服务，**Hessian协议的相关内容与HTTP基本差不多，这里就不再赘述了**。
**WebService协议特点：** 基于CXF的frontend-simple和transports-http实现，适用于系统集成，跨语言调用。 不过如非必要，强烈不推荐使用这个方式，WebService是一个相对比较重的协议传输类型，无论从性能、效率和安全性上都不太能满足微服务的需要，如果确实存在异构系统的调用，建议可以采用其他的形式。