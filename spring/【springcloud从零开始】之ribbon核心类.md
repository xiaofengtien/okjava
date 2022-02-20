Ribbon是一个客户端负载均衡器，它可以很好地控制HTTP和TCP客户端的行为。Feign已经默认使用了Ribbon(参考文章)

## 一、先来看看ribbon的几个核心类

1、IClientConfig 默认实现类DefaultClientConfigImpl，主要用来配置ribbon客户端的相关属性配置

2、ServerListUpdater默认实现类PollingServerListUpdater,主要负责动态更新服务器列表

start方法调用后会启动一个定时任务，延时1s开始执行，以每30s的时间间隔周期执行

周期时间间隔可以通过ribbon.ServerListRefreshInterval=1000或者ribbonClientName.ribbon.ServerListRefreshInterval=1000来设置

start方法的启动由DynamicServerListLoadBalancer初始化的时候执行调用

3、ServerList获取服务器列表

默认实现类ConfigurationBasedServerList,默认是从配置文件取服务器列表，这样配置[ribbonClinetName].ribbon.listOfServers=xxx,xxx

ConsulServerList引入consul作服务发现的实现类，主要负责获取注册中心的服务器列表

![img](http://www.uml.org.cn/wfw/images/202008051.png)

备注：可以通过配置来扩展自己的ServerList实现，像这样：[ribbonClient].ribbon.NIWSServerListClassName=类名

4、ServerListFilter服务器列表过滤器

默认实现类ZonePreferenceServerListFilter主要根据分区来过滤服务器列表

HealthServiceServerListFilter引入consul服务发现的实现类，主要负责过滤consul健康检查通过的服务器列表（在ConsulServerList中会获取所有的服务器列表，包括健康检查没有通过的服务器）

![img](http://www.uml.org.cn/wfw/images/202008052.png)

备注：可以通过配置来扩展自己的ServerList实现，像这样[ribbonClient].ribbon.NIWSServerListFilterClassName=类名

5、IPing检查服务器是否或者

默认实现DummyPing，这是一个假的检测着，永远返回是true

ConsulPing，引入consul服务发现的实现类，主要根据consul返回的checks参数来判断服务器是否活着，跟HealthServiceServerListFilter的过滤判断一样

![img](http://www.uml.org.cn/wfw/images/202008053.png)

备注：可以通过配置来扩展自己的ServerList实现，像这样：[ribbonClient].ribbon.NFLoadBalancerPingClassName=类名

6、IRule负载均衡选择器

默认实现ZoneAvoidanceRule，复合判断server所在区域的性能和server的可用性选择server

RandomRule：随机选择一个server

RoundRobinRule:roundRobin方式轮询选择server

RetryRule:对选定的负载均衡策略机上重试机制。

WeightedResponseTimeRule:根据响应时间分配一个weight，响应时间越长，weight越小，被选中的可能性越低。

AvailabilityFilteringRule:过滤掉那些因为一直连接失败的被标记为circuit tripped的后端server，并过滤掉那些高并发的的后端server（active connections 超过配置的阈值）

BestAvailableRule:选择一个最小的并发请求的server

![img](http://www.uml.org.cn/wfw/images/202008054.png)

7、ILoadBalancer负载均衡总控制器，默认实现类ZoneAwareLoadBalancer，其启动了整个负载均衡客户端

可以通过配置来扩展自己的ServerList实现，像这样：[ribbonClient].ribbon.NFLoadBalancerClassName=类名

![img](http://www.uml.org.cn/wfw/images/202008055.png)

负载均衡器启动时序图：

![img](http://www.uml.org.cn/wfw/images/202008056.png)

初始化时首先会初始化一个定时任务，每隔30s执行一次，缓存里面会保存所有从注册中心获取实例列表allServerList和经过ping成功的实例列表upServerList，但是负载均衡的时候拿到的列表是allServerList而非upServerList，不明白这个ping的意义在哪里

启动一个定时任务，定时从注册中心拉取服务列表，每个30s执行一次

初始化获取服务列表，拉取到的服务列表经过ServiceFilter过滤后保存在缓存里面

当客户端发起调用的时候会调用ILoadBalancer的chooseServer方法，根据IRule的负载均衡算法选择一个实例返回给调用者