源码是 2.6.5 版本

## **URL**

不过在进行服务暴露流程分析之前有必要先谈一谈 URL，有人说这 URL 和 Dubbo 啥关系？有关系，有很大的关系！

一般而言我们说的 URL 指的就是统一资源定位符，在网络上一般指代地址，本质上看其实就是一串包含特殊格式的字符串，标准格式如下：

```text
protocol://username:password@host:port/path?key=value&key=value
```

Dubbo 就是采用 URL 的方式来作为约定的参数类型，被称为公共契约，就是我们都通过 URL 来交互，来交流。

你想一下如果没有一个约束，**没有指定一个都公共的契约那么不同的接口就会以不同的参数来传递信息**，一会儿用 Map、一会儿用特定分隔的字符串，这就是导致整体很乱，并且解析不能统一。

而**用了一个统一的契约之后，那么代码就更加的规范化、形成一种统一的格式**，所有人对参数就一目了然，不用去揣测一些参数的格式等等。

而且**用 URL 作为一个公共约束充分的利用了我们对已有概念的印象，通俗易懂并且容易扩展**，我们知道 URL 要加参数只管往后面拼接就完事儿了。

因此 Dubbo 用 URL 作为配置总线，贯穿整个体系，源码中 URL 的身影无处不在。

URL 具体的参数如下：

- protocol：指的是 dubbo 中的各种协议，如：dubbo thrift http
- username/password：用户名/密码
- host/port：主机/端口
- path：接口的名称
- parameters：参数键值对

## **配置解析**

一般常用 XML 或者注解来进行 Dubbo 的配置，我稍微说一下 XML 的，这块其实是属于 Spring 的内容，我不做过多的分析，就稍微讲一下大概的原理。

Dubbo 利用了 Spring 配置文件扩展了自定义的解析，像 dubbo.xsd 就是用来约束 XML 配置时候的标签和对应的属性用的，然后 Spring 在解析到自定义的标签的时候会查找 spring.schemas 和 spring.handlers。

![img](https://pic2.zhimg.com/80/v2-7688c9e17249c316a89b4964947ce7e9_1440w.jpeg)

![img](https://pic2.zhimg.com/80/v2-f07145752a770f580966c9a44d37fc1d_1440w.jpeg)

spring.schemas 就是指明了约束文件的路径，而 spring.handlers 指明了利用该 handler 来解析标签，你看好的框架都是会预留扩展点的，讲白了就是去固定路径的固定文件名去找你扩展的东西，这样才能让用户灵活的使用。

我们再来看一下 DubboNamespaceHandler 都干了啥。

![img](https://pic4.zhimg.com/80/v2-1b0627e8ccc509fccb1c6b2149fdf59f_1440w.jpg)

讲白了就是将标签对应的解析类关联起来，这样在解析到标签的时候就知道委托给对应的解析类解析，本质就是为了生成 Spring 的 BeanDefinition，然后利用 Spring 最终创建对应的对象。

## **服务暴露全流程**

我们在深入源码之前来看下总的流程，**有个大致的印象看起来比较不容易晕**。

从**代码的流程**来看大致可以分为三个步骤（本文默认都需要暴露服务到注册中心）。

第一步是检测配置，如果有些配置空的话会默认创建，并且组装成 URL 。

第二步是暴露服务，包括暴露到本地的服务和远程的服务。

第三步是注册服务至注册中心。

![img](https://pic4.zhimg.com/80/v2-b189f6408f5b5050bf81e489f439b07f_1440w.jpg)

从**对象构建转换的角度**看可以分为两个步骤。

第一步是将服务实现类转成 Invoker。

第二部是将 Invoker 通过具体的协议转换成 Exporter。

![img](https://pic1.zhimg.com/80/v2-eac9a268d6e88641aeefed44e080fbd8_1440w.jpg)

## **服务暴露源码分析**

接下来我们进入源码分析阶段，从上面配置解析的截图标红了的地方可以看到 service 标签其实就是对应 ServiceBean，我们看下它的定义。

![img](https://pic2.zhimg.com/80/v2-7788f7c71698c1b62cb26b99e4cfde95_1440w.jpg)

这里又涉及到 Spring 相关内容了，可以看到它实现了 `ApplicationListener<ContextRefreshedEvent>`，这样就会**在 Spring IOC 容器刷新完成后调用 `onApplicationEvent` 方法，而这个方法里面做的就是服务暴露**，这就是服务暴露的启动点。

![img](https://pic2.zhimg.com/80/v2-d392ec4edd466cb32c18956d44d8a351_1440w.jpg)

可以看到，如果不是延迟暴露、并且还没暴露过、并且支持暴露的话就执行 export 方法，而 export 最终会调用父类的 export 方法，我们来看看。

![img](https://pic3.zhimg.com/80/v2-78812eae084d994a927a810b45446782_1440w.jpg)

主要就是检查了一下配置，确认需要暴露的话就暴露服务， doExport 这个方法很长，不过都是一些检测配置的过程，虽说不可或缺不过不是我们关注的重点，我们重点关注里面的 doExportUrls 方法。

![img](https://pic3.zhimg.com/80/v2-da3580d8ffb60e27a1b203c756263d56_1440w.jpg)

可以看到 Dubbo 支持多注册中心，并且支持多个协议，一个服务如果有多个协议那么就都需要暴露，比如同时支持 dubbo 协议和 hessian 协议，那么需要将这个服务用两种协议分别向多个注册中心（如果有多个的话）暴露注册。

loadRegistries 方法我就不做分析了，就是根据配置组装成注册中心相关的 URL ，我就给大家看下拼接成的 URL的样子。

```text
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&pid=7960&qos.port=22222&registry=zookeeper&timestamp=1598624821286
```

我们接下来关注的重点在 doExportUrlsFor1Protocol 方法中，这个方法挺长的，我会截取大致的部分来展示核心的步骤。

![img](https://pic4.zhimg.com/80/v2-2097013dda89e6dc2e2605eeb684016f_1440w.jpg)

此时构建出来的 URL 长这样，可以看到走得是 dubbo 协议。

![img](https://pic1.zhimg.com/80/v2-4975c8fd84953fc0fad08c8e6b8c69cc_1440w.jpg)

然后就是要根据 URL 来进行服务暴露了，我们再来看下代码，这段代码我就直接截图了，因为需要断点的解释。

![img](https://pic1.zhimg.com/80/v2-5d9c444251ebdfc913cbfa91088118cc_1440w.jpg)

### **本地暴露**

<img src="https://img2018.cnblogs.com/blog/1159846/201812/1159846-20181216204529044-31023536.png" alt="img" style="zoom:50%;" />

我们再来看一下 exportLocal 方法，这个方法是**本地暴露**，走的是 injvm 协议，可以看到它搞了个新的 URL 修改了协议。

![img](https://pic4.zhimg.com/80/v2-459af91db7660b9ebdc9a8c730c0f373_1440w.jpg)

我们来看一下这个 URL，可以看到协议已经变成了 injvm。

![img](https://pic2.zhimg.com/80/v2-92bf807e82ead0bd256bfc9dfc5cde21_1440w.jpg)

这里的 export 其实就涉及到上一篇文章讲的自适应扩展了。

```text
 Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
```

Protocol 的 export 方法是标注了 @ Adaptive 注解的，因此会生成代理类，然后代理类会根据 Invoker 里面的 URL 参数得知具体的协议，然后通过 Dubbo SPI 机制选择对应的实现类进行 export，而这个方法就会调用 InjvmProtocol#export 方法。

![img](https://pic3.zhimg.com/80/v2-494c2f3dc5d1fbae6628c2968f20d2c2_1440w.jpg)

![img](https://pic4.zhimg.com/80/v2-cbf59a66c654ccffe740ce11af13a5ff_1440w.jpg)

我们再来看看转换得到的 export 到底长什么样子。

![img](https://pic3.zhimg.com/80/v2-4b70ad89cd54eee3a8160ee7dfe6f43e_1440w.jpg)

从图中可以看到实际上就是具体实现类层层封装， invoker 其实是由 Javassist 创建的，具体创建过程 proxyFactory.getInvoker 就不做分析了，对 Javassist 有兴趣的同学自行去了解，之后可能会写一篇，至于 dubbo 为什么用 javassist 而不用 jdk 动态代理是**因为 javassist 快**。

### **为什么要封装成 invoker**

至于为什么要**封装成 invoker 其实就是想屏蔽调用的细节，统一暴露出一个可执行体**，这样调用者简单的使用它，向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

### **为什么要搞个本地暴露呢**

因为可能存在同一个 JVM 内部引用自身服务的情况，因此**暴露的本地服务在内部调用的时候可以直接消费同一个 JVM 的服务避免了网络间的通信**。

可以有些同学已经有点晕，没事我这里立马搞个图带大家过一遍。

<img src="https://pic1.zhimg.com/80/v2-3a1094b62a3b4273a51dcb0e839b08d8_1440w.jpg" alt="img" style="zoom: 33%;" />

对 exportLocal 再来一波时序图分析。

<img src="https://pic1.zhimg.com/80/v2-6e6c348f019f34b93a8f41d1013766ec_1440w.jpg" alt="img" style="zoom:50%;" />

### **远程暴露**

<img src="https://img2018.cnblogs.com/blog/1159846/201812/1159846-20181216205008489-858269681.png" alt="img" style="zoom: 67%;" />

至此本地暴露已经好了，接下来就是远程暴露了，即下面这一部分代码

![img](https://pic4.zhimg.com/80/v2-0a42552a78b3859c243fc73c8db5588f_1440w.jpeg)

也和本地暴露一样，需要封装成 Invoker ，不过这里相对而言比较复杂一些，我们先来看下 registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()) 将 URL 拼接成什么样子。

> registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo://192.168.1.17:20880/com.alibaba.dubbo.demo.DemoService....

因为很长，我就不截全了，可以看到走 registry 协议，然后参数里又有 export=dubbo://，这个走 dubbo 协议，所以我们可以得知会先通过 registry 协议找到 RegistryProtocol 进行 export，并且在此方法里面还会根据 export 字段得到值然后执行 DubboProtocol 的 export 方法。

**大家要挺住，就快要完成整个流程的解析了！**

现在我们把目光聚焦到 RegistryProtocol#export 方法上，我们先过一遍整体的流程，然后再进入 doLocalExport 的解析。

![img](https://pic4.zhimg.com/80/v2-c6a6fa1e08a1b1ac5924cc7224e46617_1440w.jpg)

可以看到这一步主要是将上面的 export=dubbo://... 先转换成 exporter ，然后获取注册中心的相关配置，如果需要注册则向注册中心注册，并且在 ProviderConsumerRegTable 这个表格中记录服务提供者，其实就是往一个 ConcurrentHashMap 中将塞入 invoker，key 就是服务接口全限定名，value 是一个 set，set 里面会存包装过的 invoker 。

![img](https://pic3.zhimg.com/80/v2-7d0e3d64c2163395ba1323297974e4ee_1440w.jpg)

我们再把目光聚焦到 doLocalExport 方法内部。

![img](https://pic3.zhimg.com/80/v2-5305fb8157d963b7075462d26d47b986_1440w.jpg)

这个方法没什么难度，主要就是根据URL上 Dubbo 协议暴露出 exporter，接下来就看下 DubboProtocol#export 方法。

![img](https://pic2.zhimg.com/80/v2-fe8a59fcb54ba6738af064e93a22a14d_1440w.jpg)

可以看到这里的关键其实就是打开 Server ，RPC 肯定需要远程调用，这里我们用的是 NettyServer 来监听服务。

![img](https://pic2.zhimg.com/80/v2-5be02d889ce48385553ba8747ceb91f9_1440w.jpg)

再下面我就不跟了，我总结一下 Dubbo 协议的 export 主要就是根据 URL 构建出 key（例如有分组、接口名端口等等），然后 key 和 invoker 关联，关联之后存储到 DubboProtocol 的 exporterMap 中，然后如果是服务初次暴露则会创建监听服务器，默认是 NettyServer，并且会初始化各种 Handler 比如心跳啊、编解码等等。

看起来好像流程结束了？并没有， Filter 到现在还没出现呢？有隐藏的措施，上一篇 Dubbo SPI 看的仔细的各位就知道在哪里触发的。

其实上面的 protocol 是个代理类，在内部会通过 SPI 机制找到具体的实现类。

![img](https://pic1.zhimg.com/80/v2-86f6837feb970d13d261b43f52da571c_1440w.jpg)

这张图是上一篇文章的，可以看到 export 具体的实现。

![img](https://pic2.zhimg.com/80/v2-cb2f74f94305940d8918198889d22bb9_1440w.jpg)

复习下上一篇的要点，通过 Dubbo SPI 扫包会把 wrapper 结尾的类缓存起来，然后当加载具体实现类的时候会包装实现类，来实现 Dubbo 的 AOP，我们看到 DubboProtocol 有什么包装类。

![img](https://pic4.zhimg.com/80/v2-178380f0b80c99020c9af57afdc2a293_1440w.jpg)

可以看到有两个，分别是 ProtocolFilterWrapper 和 ProtocolListenerWrapper

对于所有的 Protocol 实现类来说就是这么个调用链。

![img](https://pic4.zhimg.com/80/v2-eae6596d560e738643ca27b2a97de1e7_1440w.jpeg)

而在 ProtocolFilterWrapper 的 export 里面就会把 invoker 组装上各种 Filter。

![img](https://pic1.zhimg.com/80/v2-5958b8cafa329d561c0b725458de0c30_1440w.jpg)

看看有 8 个在。

![img](https://pic4.zhimg.com/80/v2-64353e9368b207e57095e088cb102917_1440w.jpg)

我们再来看下 zookeeper 里面现在是怎么样的，关注 dubbo 目录。

![img](https://pic3.zhimg.com/80/v2-f907beefef48d36c878963481c3acfae_1440w.jpg)

两个 service 占用了两个目录，分别有 configurators 和 providers 文件夹，文件夹里面记录的就是 URL 的那一串，值是服务提供者 ip。

至此服务流程暴露差不多完结了，可以看到还是有点内容在里面的，并且还需要掌握 Dubbo SPI，不然有些点例如自适应什么的还是很难理解的。最后我再来一张完整的流程图带大家再过一遍，具体还是有很多细节，不过不是主干我就不做分析了，不然文章就有点散。

<img src="https://pic1.zhimg.com/80/v2-a6b42dbe67089315b61ee36f586ab094_1440w.jpg" alt="img" style="zoom: 50%;" />

然后再引用一下官网的时序图。

![img](https://pic3.zhimg.com/80/v2-26195b0390e841e4f39db0a2fa32ee62_1440w.jpg)

## **总结**

服务暴露的过程起始于 Spring IOC 容器刷新完成之时，具体的流程就是根据配置得到 URL，再利用 Dubbo SPI 机制根据 URL 的参数选择对应的实现类，实现扩展。

通过 javassist 动态封装 ref (你写的服务实现类)，统一暴露出 Invoker 使得调用方便，屏蔽底层实现细节，然后封装成 exporter 存储起来，等待消费者的调用，并且会将 URL 注册到注册中心，使得消费者可以获取服务提供者的信息。

```java
Service->Invoker->Exporter
```

