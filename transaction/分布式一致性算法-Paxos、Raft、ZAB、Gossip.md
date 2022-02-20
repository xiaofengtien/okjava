## 分布式一致性算法-Paxos、Raft、ZAB、Gossip

### 为什么需要一致性

1. 数据不能存在单个节点（主机）上，否则可能出现单点故障。
2. 多个节点（主机）需要保证具有相同的数据。
3. 一致性算法就是为了解决上面两个问题。

### 一致性算法的定义

一致性就是数据保持一致，在分布式系统中，可以理解为多个节点中数据的值是一致的。

### 一致性的分类

#### 强一致性

保证系统改变提交以后立即改变集群的状态。

- 2PC
- 3PC
- Paxos
- Raft（muti-paxos）
- ZAB（muti-paxos）

#### 弱一致性

也叫最终一致性，系统不保证改变提交以后立即改变集群的状态，但是随着时间的推移最终状态是一致的。

- DNS系统
- Gossip协议

### 一致性算法实现举例

- Google的Chubby分布式锁服务，采用了Paxos算法
- etcd分布式键值数据库，采用了Raft算法
- ZooKeeper分布式应用协调服务，Chubby的开源实现，采用ZAB算法

### 2PC（两阶段提交）

两阶段提交是一种保证分布式系统数据一致性的协议，现在很多数据库都是采用的两阶段提交协议来完成 **分布式事务** 的处理。

在介绍2PC之前，我们先来想想分布式事务到底有什么问题呢？

还拿秒杀系统的下订单和加积分两个系统来举例吧（我想你们可能都吐了🤮🤮🤮），我们此时下完订单会发个消息给积分系统告诉它下面该增加积分了。如果我们仅仅是发送一个消息也不收回复，那么我们的订单系统怎么能知道积分系统的收到消息的情况呢？如果我们增加一个收回复的过程，那么当积分系统收到消息后返回给订单系统一个 `Response` ，但在中间出现了网络波动，那个回复消息没有发送成功，订单系统是不是以为积分系统消息接收失败了？它是不是会回滚事务？但此时积分系统是成功收到消息的，它就会去处理消息然后给用户增加积分，这个时候就会出现积分加了但是订单没下成功。

所以我们所需要解决的是在分布式系统中，整个调用链中，我们所有服务的数据处理要么都成功要么都失败，即所有服务的 **原子性问题** 。

在两阶段提交中，主要涉及到两个角色，分别是协调者和参与者。

第一阶段：当要执行一个分布式事务的时候，事务发起者首先向协调者发起事务请求，然后协调者会给所有参与者发送 `prepare` 请求（其中包括事务内容）告诉参与者你们需要执行事务了，如果能执行我发的事务内容那么就先执行但不提交，执行后请给我回复。然后参与者收到 `prepare` 消息后，他们会开始执行事务（但不提交），并将 `Undo` 和 `Redo` 信息记入事务日志中，之后参与者就向协调者反馈是否准备好了。

第二阶段：第二阶段主要是协调者根据参与者反馈的情况来决定接下来是否可以进行事务的提交操作，即提交事务或者回滚事务。

比如这个时候 **所有的参与者** 都返回了准备好了的消息，这个时候就进行事务的提交，协调者此时会给所有的参与者发送 **`Commit` 请求** ，当参与者收到 `Commit` 请求的时候会执行前面执行的事务的 **提交操作** ，提交完毕之后将给协调者发送提交成功的响应。

而如果在第一阶段并不是所有参与者都返回了准备好了的消息，那么此时协调者将会给所有参与者发送 **回滚事务的 `rollback` 请求**，参与者收到之后将会 **回滚它在第一阶段所做的事务处理** ，然后再将处理情况返回给协调者，最终协调者收到响应后便给事务发起者返回处理失败的结果。

![2PC流程](https://img-blog.csdnimg.cn/img_convert/7ce4e40b68d625676bb42c29efce046a.png)

个人觉得 2PC 实现得还是比较鸡肋的，因为事实上它只解决了各个事务的原子性问题，随之也带来了很多的问题。

* **单点故障问题**，如果协调者挂了那么整个系统都处于不可用的状态了。
* **阻塞问题**，即当协调者发送 `prepare` 请求，参与者收到之后如果能处理那么它将会进行事务的处理但并不提交，这个时候会一直占用着资源不释放，如果此时协调者挂了，那么这些资源都不会再释放了，这会极大影响性能。
* **数据不一致问题**，比如当第二阶段，协调者只发送了一部分的 `commit` 请求就挂了，那么也就意味着，收到消息的参与者会进行事务的提交，而后面没收到的则不会进行事务提交，那么这时候就会产生数据不一致性问题。

###  3PC（三阶段提交）

因为2PC存在的一系列问题，比如单点，容错机制缺陷等等，从而产生了 **3PC（三阶段提交）** 。那么这三阶段又分别是什么呢？

> 千万不要吧PC理解成个人电脑了，其实他们是 phase-commit 的缩写，即阶段提交。

1. **CanCommit阶段**：协调者向所有参与者发送 `CanCommit` 请求，参与者收到请求后会根据自身情况查看是否能执行事务，如果可以则返回 YES 响应并进入预备状态，否则返回 NO 。
2. **PreCommit阶段**：协调者根据参与者返回的响应来决定是否可以进行下面的 `PreCommit` 操作。如果上面参与者返回的都是 YES，那么协调者将向所有参与者发送 `PreCommit` 预提交请求，**参与者收到预提交请求后，会进行事务的执行操作，并将 `Undo` 和 `Redo` 信息写入事务日志中** ，最后如果参与者顺利执行了事务则给协调者返回成功的响应。如果在第一阶段协调者收到了 **任何一个 NO** 的信息，或者 **在一定时间内** 并没有收到全部的参与者的响应，那么就会中断事务，它会向所有参与者发送中断请求（abort），参与者收到中断请求之后会立即中断事务，或者在一定时间内没有收到协调者的请求，它也会中断事务。
3. **DoCommit阶段**：这个阶段其实和 `2PC` 的第二阶段差不多，如果协调者收到了所有参与者在 `PreCommit` 阶段的 YES 响应，那么协调者将会给所有参与者发送 `DoCommit` 请求，**参与者收到 `DoCommit` 请求后则会进行事务的提交工作**，完成后则会给协调者返回响应，协调者收到所有参与者返回的事务提交成功的响应之后则完成事务。若协调者在 `PreCommit` 阶段 **收到了任何一个 NO 或者在一定时间内没有收到所有参与者的响应** ，那么就会进行中断请求的发送，参与者收到中断请求后则会 **通过上面记录的回滚日志** 来进行事务的回滚操作，并向协调者反馈回滚状况，协调者收到参与者返回的消息后，中断事务。

![3PC流程](https://img-blog.csdnimg.cn/img_convert/d0b44361c746593f70a6e42c298b413a.png)

> 这里是 `3PC` 在成功的环境下的流程图，你可以看到 `3PC` 在很多地方进行了超时中断的处理，比如协调者在指定时间内为收到全部的确认消息则进行事务中断的处理，这样能 **减少同步阻塞的时间** 。还有需要注意的是，**`3PC` 在 `DoCommit` 阶段参与者如未收到协调者发送的提交事务的请求，它会在一定时间内进行事务的提交**。为什么这么做呢？是因为这个时候我们肯定**保证了在第一阶段所有的协调者全部返回了可以执行事务的响应**，这个时候我们有理由**相信其他系统都能进行事务的执行和提交**，所以**不管**协调者有没有发消息给参与者，进入第三阶段参与者都会进行事务的提交操作。

总之，`3PC` 通过一系列的超时机制很好的缓解了阻塞问题，但是最重要的一致性并没有得到根本的解决，比如在 `PreCommit` 阶段，当一个参与者收到了请求之后其他参与者和协调者挂了或者出现了网络分区，这个时候收到消息的参与者都会进行事务提交，这就会出现数据不一致性问题

### Paxos算法

#### 概念介绍

1. Proposal提案，即分布式系统的修改请求，可以表示为[提案编号N，提案内容value]
2. Client用户，类似社会民众，负责提出建议
3. Propser议员，类似基层人大代表，负责帮Client上交提案
4. Acceptor投票者，类似全国人大代表，负责为提案投票，不同意比自己以前接收过的提案编号要小的提案，其他提案都同意，例如A以前给N号提案表决过，那么再收到小于等于N号的提案时就直接拒绝了
5. Learner提案接受者，类似记录被通过提案的记录员，负责记录提案

#### Basic Paxos算法

步骤：

1. Propser准备一个N号提案
2. Propser询问Acceptor中的多数派是否接收过N号的提案，如果都没有进入下一步，否则本提案不被考虑
3. Acceptor开始表决，Acceptor无条件同意从未接收过的N号提案，达到多数派同意后，进入下一步
4. Learner记录提案

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5Ufe8hmYbTgl3YB8zPpcZsHjLSgAoNibNicAx835y2VbUI6ondeMk7P5Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### 节点故障

- 若Proposer故障，没关系，再从集群中选出Proposer即可
- 若Acceptor故障，表决时能达到多数派也没问题

##### 潜在问题-活锁

假设系统有多个Proposer，他们不断向Acceptor发出提案，还没等到上一个提案达到多数派下一个提案又来了，就会导致Acceptor放弃当前提案转向处理下一个提案，于是所有提案都别想通过了。

#### Multi Paxos算法

根据Basic Paxos的改进：整个系统只有一个Proposer，称之为Leader。

步骤：

1. 若集群中没有Leader，则在集群中选出一个节点并声明它为第M任Leader。
2. 集群的Acceptor只表决最新的Leader发出的最新的提案
3. 其他步骤和Basic Paxos相同

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5OIr5IEk3d4x9XpjCoT2HP9KlTRT87ALre5oekVPUH9JKy5Ik5ZHvhQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### 算法优化

Multi Paxos角色过多，对于计算机集群而言，可以将Proposer、Acceptor和Learner三者身份**集中在一个节点上**，此时只需要从集群中选出Proposer，其他节点都是Acceptor和Learner，这就是接下来要讨论的Raft算法

### Raft算法

Paxos算法不容易实现，Raft算法是对Paxos算法的简化和改进

#### 概念介绍

1. Leader总统节点，负责发出提案
2. Follower追随者节点，负责同意Leader发出的提案
3. Candidate候选人，负责争夺Leader

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5cCTLoYyWrkxYYiaia0oczV61KJ462PtO405PYiaiaIINBr6R7iczXB8G7Yw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Raft算法将一致性问题分解为两个的子问题，**Leader选举**和**状态复制**

#### Leader选举

1. 每个Follower都持有一个定时器

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5k90SW3vlfbSSd10bw3grCR1rae0jORqWCV1ZzAonTGnS8mp97hP0XA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 当定时器时间到了而集群中仍然没有Leader，Follower将声明自己是Candidate并参与Leader选举，同时**将消息发给其他节点来争取他们的投票**，若其他节点长时间没有响应Candidate将重新发送选举信息

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF58CqLjfBczoYcJiaCkIdNPZEEHKE4Fib5Saibbfh9g824V6vcLUMqFEbSQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 集群中其他节点将给Candidate投票

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF543awwjykZcodI3d9xTibomYzN9YYECepgYIvl4AAoMx8BStibcY3MSpg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 获得多数派支持的Candidate将成为第M任Leader（M任是最新的任期）

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5picpqWyGkBVkQ9xwviaDYkk56HY5mhO07V7XJB5c1caUtYcf8vXKwRFA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 在任期内的Leader会**不断发送心跳**给其他节点证明自己还活着，其他节点受到心跳以后就清空自己的计时器并回复Leader的心跳。这个机制保证其他节点不会在Leader任期内参加Leader选举。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5t9gyDk6MGFn4Z7dXS3zyicL9KP5zYO2krU5HYOIxh4CSDtia24EibUBug/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5JSQfmzbYrQ3Xd0ENQwp3JRXmUcumaaFx5RgDZyqqiak7ib27KPZhRfXg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 当Leader节点出现故障而导致Leader失联，没有接收到心跳的Follower节点将准备成为Candidate进入下一轮Leader选举
2. 若出现两个Candidate同时选举并获得了相同的票数，那么这两个Candidate将随机推迟一段时间后再向其他节点发出投票请求，这保证了再次发送投票请求以后不冲突

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5I3ZoEWosKMBxz9xcuDkFia174HPrdCDfwdNCFDQ2AE2lkAtoLb5Br3g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 状态复制

1. Leader负责接收来自Client的提案请求（**红色提案表示未确认**）

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5Whwxmzibia8JFlvwSebv78vakR7XJb67yuEZPgbDiaAZXA4BqSnEfy1zg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 提案内容将包含在Leader发出的下一个心跳中

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5dfa55jDKbibtkjjSaqQr8ewJRFeIA3NwNIectNPNTKjdAmHVEZaSNcg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. Follower接收到心跳以后回复Leader的心跳

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5X9s9JUJiaj0TujpkibgYRYr5v7OtPLWtoCbAriadY2ltAibkSBHg0dhQ2w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. Leader接收到多数派Follower的回复以后**确认提案**并写入自己的存储空间中并**回复Client**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF50teOIHXfxjv7z1JAJZ4MuiaNjxyM49ob2zP9h5sPWticibb34RRhCUZqQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. **Leader通知Follower**节点确认提案并写入自己的存储空间，随后所有的节点都拥有相同的数据

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5KErK0OVkOCBsZAkrY7hcKqdPde1TwrotK5djJXKC3iaJ0cSvNS1iareA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 若集群中出现网络异常，导致集群被分割，将出现多个Leader

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF58EWJ2ozeOXhKw5TxUu4ZzVFcGibSrWxqgB47Vuu9mDK04yvlNqEsD6w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 被分割出的非多数派集群将无法达到共识，即**脑裂**，如图中的A、B节点将无法确认提案

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5nt3Pa9rWhl7lVDh4Wx2Imiawd3SyicHVpFF4X88oicbwTuy64zsZGzZ8w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5kF0oIjzMgaJWCNu9brkNabsGBjW3U19ichpgRicUrlFzj7m2vYel3CgQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 当集群再次连通时，将只听从最新任期Leader的指挥，旧Leader将退化为Follower，如图中B节点的Leader（任期1）需要听从D节点的Leader（任期2）的指挥，此时集群重新达到一致性状态

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5L0hOfkEyIIlqFBPQUCZdYHNHf3okSBJ37BgMps9VSRCLGaEtS5QcnA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF59Up7QydeQf7camU31XUjIezeumVoDeW64Zqh47JAXCHRGSTG88ed5Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### ZAB算法

ZAB也是对Multi Paxos算法的改进，大部分和raft相同,和raft算法的主要区别：

1. 对于Leader的任期，raft叫做term，而ZAB叫做epoch
2. 在状态复制的过程中，raft的心跳从Leader向Follower发送，而ZAB则相反。

### Gossip算法

Gossip算法每个节点都是对等的，即没有角色之分。Gossip算法中的每个节点都会将数据改动告诉其他节点（类似传八卦）。有话说得好："最多通过六个人你就能认识全世界任何一个陌生人"，因此数据改动的消息很快就会传遍整个集群。

1. 集群启动，如下图所示（这里设置集群有20个节点）

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5LLbD2gCSlyu1nJt2DIFMFhLhYTeJT1WebMiabEv13hdBfGsPqRpB1Jg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 某节点收到数据改动，并将改动传播给其他4个节点，传播路径表示为较粗的4条线

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5SyRDp00cs0RunNt2Mob2gNiadJs1J9rgibYa6VXdPbcs5hgW5smraxmg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Baq5lYpIw7XBndbkcpyeQun4CYcY2PF5O6Wc3G4DO9SibhFQLTEzOkfYtOhsL5uXKUfib15rPj9qaibThibHmPKibvQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 收到数据改动的节点重复上面的过程直到所有的节点都被感染

