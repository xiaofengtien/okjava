## 附录

1. TCP 连接状态的查询
2. MSL 时间
3. TCP 三次握手和四次握手

### 附录 A：查询 TCP 连接状态

Mac 下，查询 TCP 连接状态的具体命令：

```sh
// Mac 下，查询 TCP 连接状态
$ netstat -nat |grep TIME_WAIT

// Mac 下，查询 TCP 连接状态，其中 -E 表示 grep 或的匹配逻辑
$ netstat -nat | grep -E "TIME_WAIT|Local Address"
Proto  Recv-Q Send-Q Local  Address  Foreign  Address  (state)
tcp4 0  0  127.0.0.1.1080  127.0.0.1.59061 TIME_WAIT

// 统计：各种连接的数量
`$ netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
`ESTABLISHED 1154
`TIME_WAIT 1645
```

### 附录 B：MSL 时间

MSL，Maximum Segment Lifetime，“报文最大生存时间”，

1. 任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。（IP 报文）
2. TCP报文 （segment）是ip数据报（datagram）的数据部分。

Tips：

> RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。

2MSL，TCP 的 `TIME_WAIT` 状态，也称为**2MSL等待状态**：

1. [当TCP的一端发起主动关闭（收到 FIN 请求），在发出最后一个ACK 响应后，即第3次握 手完成后，发送了第四次握手的ACK包后，就进入了TIME_WAIT状态。]
2. [必须在此状态上停留两倍的MSL时间，等待2MSL时间主要目的是怕最后一个 ACK包对方没收到，那么对方在超时后将重发第三次握手的FIN包，主动关闭端接到重发的FIN包后，可以再发一个ACK应答包。]
3. [在 TIME_WAIT 状态时，两端的端口不能使用，要等到2MSL时间结束，才可继续使用。（IP 层）]
4. [当连接处于2MSL等待阶段时，任何迟到的报文段都将被丢弃。]

[不过在实际应用中，可以通过设置 「**SO_REUSEADDR选项**」，达到不必等待2MSL时间结束，即可使用被占用的端口。]

### 附录 C：TCP 三次握手和四次握手

详细细节，参考：

- TCP的三次握手与四次挥手（详解+动图）

具体示意图：

1. **三次握手**，`建立`连接过程
2. **四次挥手**，`释放`连接过程

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpROmJ9rmCxguKX6rM6tSp67v1l80fl15rONnQg3zU7UCr5Lic7ic07A2QW7rE08cVj951LUTgMXTTkw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

[几个核心疑问：]

[1、 **time_wait 是「服务器端」的状态？or 「客户端」的状态？**]

- [RE：time_wait 是「主动关闭 TCP 连接」一方的状态，可能是「客服端」的，也可能是「服务器端」的]
- [一般情况下，都是「客户端」所处的状态；「服务器端」一般设置「不主动关闭连接」]

[2、 **服务器在对外服务时，是「客户端」发起的断开连接？还是「服务器」发起的断开连接？**]

- 正常情况下，都是「客户端」发起的断开连接
- [「服务器」一般设置为「不主动关闭连接」，服务器通常执行「被动关闭」]
- [但 HTTP 请求中，http 头部 connection 参数，可能设置为 close，则，服务端处理完请求会主动关闭 TCP 连接]

[关于 Apache httpd 服务器的关联配置，参考：]

> https://elf8848.iteye.com/blog/1739571

[关于 HTTP 请求中，设置的主动关闭 TCP 连接的机制：TIME_WAIT的是主动断开方才会出现的，所以主动断开方是服务端？]

- 答案是是的。在HTTP1.1协议中，有个 Connection 头，Connection有两个值，close和keep-alive，这个头就相当于客户端告诉服务端，服务端你执行完成请求之后，是关闭连接还是保持连接，保持连接就意味着在保持连接期间，只能由客户端主动断开连接。还有一个keep-alive的头，设置的值就代表了服务端保持连接保持多久。]
- HTTP默认的Connection值为close，那么就意味着关闭请求的一方几乎都会是由服务端这边发起的。那么这个服务端产生TIME_WAIT过多的情况就很正常了。
- 虽然HTTP默认Connection值为close，但是，现在的浏览器发送请求的时候一般都会设置Connection为keep-alive了。所以，也有人说，现在没有必要通过调整参数来使TIME_WAIT降低了。

[关于 **time_wait**：]

1、TCP 连接建立后，「**主动关闭连接**」的一端，收到对方的 FIN 请求后，发送 ACK 响应，会处于 time_wait 状态

2、 **time_wait 状态**，存在的`必要性`：

- **可靠的实现 TCP 全双工连接的终止**：四次挥手关闭 TCP 连接过程中，最后的 ACK 是由「主动关闭连接」的一端发出的，如果这个 ACK 丢失，则，对方会重发 FIN 请求，因此，在「主动关闭连接」的一段，需要维护一个 time_wait 状态，处理对方重发的 FIN 请求；
- **处理延迟到达的报文**：由于路由器可能抖动，TCP 报文会延迟到达，为了避免「延迟到达的 TCP 报文」被误认为是「新 TCP 连接」的数据，则，需要在允许新创建 TCP 连接之前，保持一个不可用的状态，等待所有延迟报文的消失，一般设置为 2 倍的 MSL（报文的最大生存时间），解决「延迟达到的 TCP 报文」问题