# Linux下性能压测工具WRK的使用

## wrk介绍 

​	wrk 是一款针对 Http 协议的基准测试工具，它能够在单机多核 CPU 的条件下，使用系统自带的高性能 I/O 机制，如 epoll，kqueue 等，通过多线程和事件模式，对目标机器产生大量的负载

### Ubuntu/Debin安装：

```kotlin
sudo apt-get install build-essential libssl-dev git -y
git clone https://github.com/wg/wrk.git wrk
cd wrk
make
# 将可执行文件移动到 /usr/local/bin 位置
sudo cp wrk /usr/local/bin
```

### Rethat/Centos下安装：

```kotlin
sudo yum groupinstall 'Development Tools'
sudo yum install -y openssl-devel git 
git clone https://github.com/wg/wrk.git wrk
cd wrk
make
# 将可执行文件移动到 /usr/local/bin 位置
sudo cp wrk /usr/local/bin
```

### 验证安装是否成功：wrk -v

#### wrk命令行参数翻译：wrk --help

```kotlin
使用方法: wrk <选项> <被测HTTP服务的URL>                            
  Options:                                            
    -c, --connections <N>  跟服务器建立并保持的TCP连接数量  
    -d, --duration    <T>  压测时间           
    -t, --threads     <N>  使用多少个线程进行压测   
                                                      
    -s, --script      <S>  指定Lua脚本路径       
    -H, --header      <H>  为每一个HTTP请求添加HTTP头      
        --latency          在压测结束后，打印延迟统计信息   
        --timeout     <T>  超时时间     
    -v, --version          打印正在使用的wrk的详细版本信息                                                  
  <N>代表数字参数，支持国际单位 (1k, 1M, 1G)
  <T>代表时间参数，支持时间单位 (2s, 2m, 2h)

```

### wrk使用方法

wrk -t12 -c400 -d30s http://10.20.80.133/

这条命令表示，利用 wrk 对 http://10.20.80.133/ 发起压力测试，线程数为 12，模拟 400 个并发请求，持续 30 秒

### 输出压测报告

wrk -t12 -c400 -d30s --latency http://10.20.80.133/

#### 各项指标代表的含义

```kotlin
Running 30s test @ http://10.20.80.133 （压测时间30s）
  12 threads and 400 connections （共12个测试线程，400个连接）
			  （平均值） （标准差）  （最大值）（正负一个标准差所占比例）
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    （延迟）
    Latency   386.32ms  380.75ms   2.00s    86.66%
    (每秒请求数)
    Req/Sec    17.06     13.91   252.00     87.89%
  Latency Distribution （延迟分布）
     50%  218.31ms
     75%  520.60ms
     90%  955.08ms
     99%    1.93s 
  4922 requests in 30.06s, 73.86MB read (30.06s内处理了4922个请求，耗费流量73.86MB)
  Socket errors: connect 0, read 0, write 0, timeout 311 (发生错误数)
Requests/sec:    163.76 (QPS 163.76,即平均每秒处理请求数为163.76)
Transfer/sec:      2.46MB (平均每秒流量2.46MB)
```

**标准差**啥意思？标准差如果太大说明样本本身离散程度比较高，有可能系统性能波动较大

### 使用Lua脚本进行压测

在进行post请求时，通过编写 Lua 脚本的方式，在运行压测命令时，通过参数 `--script` 来指定 Lua 脚本，来满足个性化需求

wrk 支持在三个阶段对压测进行个性化，分别是启动阶段、运行阶段和结束阶段。每个测试线程，都拥有独立的Lua 运行环境。

启动阶段：

```kotlin
function setup(thread)
```

在脚本文件中实现 setup 方法，wrk 就会在测试线程已经初始化，但还没有启动的时候调用该方法。wrk会为每一个测试线程调用一次 setup 方法，并传入代表测试线程的对象 thread 作为参数。setup 方法中可操作该 thread 对象，获取信息、存储信息、甚至关闭该线程。

```lua
thread.addr             - get or set the thread's server address
thread:get(name)        - get the value of a global in the thread's env
thread:set(name, value) - set the value of a global in the thread's env
thread:stop()           - stop the thread
```

运行阶段：

```lua
function init(args)
function delay()
function request()
function response(status, headers, body)
```

- `init(args)`: 由测试线程调用，只会在进入运行阶段时，调用一次。支持从启动 wrk 的命令中，获取命令行参数；
- `delay()`： 在每次发送请求之前调用，如果需要定制延迟时间，可以在这个方法中设置；
- `request()`: 用来生成请求, 每一次请求都会调用该方法，所以注意不要在该方法中做耗时的操作；
- `response(status, headers, body)`: 在每次收到一个响应时被调用，为提升性能，如果没有定义该方法，那么wrk不会解析 `headers` 和 `body`；

结束阶段：

```kotlin
function done(summary, latency, requests)
```

done() 方法在整个测试过程中只会被调用一次，我们可以从给定的参数中，获取压测结果，生成定制化的测试报告。

自定义lua脚本中可访问的变量及方法：

wrk变量：

```lua
wrk = {
    scheme  = "http",
   host    = "localhost",
    port    = 8080,
    method  = "GET",
    path    = "/",
    headers = {},
    body    = nil,
    thread  = <userdata>,
  }
```

以上定义了一个 `table` 类型的全局变量，修改该 `wrk` 变量，会影响所有请求。

方法：

1. `wrk.fomat`
2. `wrk.lookup`
3. `wrk.connect`

上面三个方法解释如下

```lua
function wrk.format(method, path, headers, body)
    wrk.format returns a HTTP request string containing the passed parameters
    merged with values from the wrk table.
    # 根据参数和全局变量 wrk，生成一个 HTTP rquest 字符串。
function wrk.lookup(host, service)
    wrk.lookup returns a table containing all known addresses for the host
    and service pair. This corresponds to the POSIX getaddrinfo() function.
    # 给定 host 和 service（port/well known service name），返回所有可用的服务器地址信息。
function wrk.connect(addr)
    wrk.connect returns true if the address can be connected to, otherwise
    it returns false. The address must be one returned from wrk.lookup().
    # 测试给定的服务器地址信息是否可以成功创建连接
```

六、lua脚本实战训练：

1、上传：urlencoded类型的post请求：

lua脚本中的内容如下：

```kotlin
wrk.method = "POST"
wrk.body = "foo=bar&baz=quux"
wrk.headers["Content-Type"] = "application/x-www-form-urlencoded"
request = function()
  uid = math.random(1,10000000)
  path = "/test?uid="..uid
  return wrk.format(nil,path)
end
```

在 request 方法中，随机生成 1~10000000 之间的 uid，并动态生成请求 URL

执行命令： wrk -t 12 -c 300 -d 60s --script posturlencoded.lua http://10.20.80.133/

![img](https://img-blog.csdnimg.cn/20200511153048496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2d1ZmVuY2hlbg==,size_16,color_FFFFFF,t_70)

每次请求延时10s，lua脚本内容如下：

![img](https://img-blog.csdnimg.cn/20200511153254129.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2d1ZmVuY2hlbg==,size_16,color_FFFFFF,t_70)

效果如下：

![img](https://img-blog.csdnimg.cn/20200511153455817.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2d1ZmVuY2hlbg==,size_16,color_FFFFFF,t_70)

如果请求接口需要token如何发送：

```lua
token = nil
path  = "/auth"
request = function()
return wrk.format("GET", path)
end
response = function(status, headers, body)
 if not token and status == 200 then
 token = headers["X-Token"]
 path  = "/test"
wrk.headers["X-Token"] = token
   end
end
```

上面的脚本表示，在 token 为空的情况下，先请求 `/auth` 接口来认证，获取 token, 拿到 token 以后，将 token 放置到请求头中，再请求真正需要压测的 `/test` 接口。

**压测支持 HTTP pipeline 的服务：**

```lua
init = function(args)
   local r = {}
   r[1] = wrk.format(nil, "/?foo")
   r[2] = wrk.format(nil, "/?bar")
   r[3] = wrk.format(nil, "/?baz")
   req = table.concat(r)
end
request = function()
   return req
end
```

通过在 init 方法中将三个 HTTP请求拼接在一起，实现每次发送三个请求，以使用 HTTP pipeline。

lua脚本如下：

```lua
wrk.method = "POST"
wrk.body   = "foo=bar&baz=quux"
wrk.headers["Content-Type"]="application/x-www-form-urlencoded"
init = function(args)
   local r = {}
   r[1] = wrk.format(nil, "/?foo")
   r[2] = wrk.format(nil, "/?bar")
   r[3] = wrk.format(nil, "/?baz")
   req = table.concat(r)
end
request = function()
   return req
end
```

![img](https://img-blog.csdnimg.cn/202005111555158.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2d1ZmVuY2hlbg==,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20200511160335140.png)

发送json请求:

lua脚本内容如下：

```lua
wrk.method = "POST"
wrk.body = '{"userId":"1001","coinType":"GT","type":"2","amount":"5.1"}'
wrk.headers["Content-Type"]="application/json"
function request()
  return wrk.format('POST',nil,nil,body)
end
```

![img](https://img-blog.csdnimg.cn/20200511161446696.png)

发送xml格式的报文:

```lua
wrk.method = "POST"
wrk.body  =[[
<?xml version="1.0" encoding = "UTF-8"?>
<COM>
<REQ name="哈哈哈">
<USER_ID>yoyoketang</USER_ID>
<COMMODITY_ID>123456</COMMODITY_ID>
<SESSION_ID>absbnmasbnfmasbm1213</SESSION_ID>
</REQ>
</COM>
]]
wrk.headers["Content-Type"]= "text/xml"
request = function()
  return wrk.format('POST',nil,nil)
end
```

![img](https://img-blog.csdnimg.cn/20200511162917508.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2d1ZmVuY2hlbg==,size_16,color_FFFFFF,t_70)