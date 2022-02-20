创建线程有几种方式？这个问题的答案应该是可以脱口而出的吧

继承 Thread 类实现 Runnable 接口但这两种方式创建的线程是属于”三无产品“：

没有参数没有返回值没办法抛出异常

![img](https://pics7.baidu.com/feed/faedab64034f78f00c82063719f7f853b1191cb2.jpeg?token=32ce63e6ee09d9fc1f006363528adeb1)

Runnable 接口是 JDK1.0 的核心产物

![img](https://pics4.baidu.com/feed/0ff41bd5ad6eddc4e0eb5560591d44fb5366336e.jpeg?token=bfa16a735269202fbf918e4c0464374a)

用着 “三无产品” 总是有一些弊端，其中没办法拿到返回值是最让人不能忍的，于是 Callable 就诞生了

**Callable**

又是 Doug Lea 大师，又是 Java 1.5 这个神奇的版本

![img](https://pics7.baidu.com/feed/b90e7bec54e736d17ca36205fa96bdc4d462695c.jpeg?token=71ca46ea0cc875bc1d4f61e169f3c2d4)

Callable 是一个泛型接口，里面只有一个 call() 方法，**该方法可以返回泛型值 V**，使用起来就像这样：

![img](https://pics7.baidu.com/feed/9922720e0cf3d7ca0875712593d94c0f6a63a92b.jpeg?token=22724d2ab9cd773634d4ea99deb53b46)

二者都是函数式接口，里面都仅有一个方法，使用上又是如此相似，除了有无返回值，Runnable 与 Callable 就点差别吗？

**Runnable VS Callable**

两个接口都是用于多线程执行任务的，但他们还是有很明显的差别的

**执行机制**

先从执行机制上来看，Runnable 你太清楚了，它既可以用在 Thread 类中，也可以用在 **ExecutorService**类中配合线程池的使用；**Bu～～～～t， Callable 只能在 \**ExecutorService\** 中使用**，你翻遍 Thread 类，也找不到Callable 的身影

![img](https://pics3.baidu.com/feed/d833c895d143ad4bc961f372e0c4a8a9a50f0605.jpeg?token=68fec8b1c7e7dd2c003f7b00951cab56)

**异常处理**

Runnable 接口中的 run 方法签名上没有 **throws**，自然也就没办法向上传播受检异常；而 Callable 的 call() 方法签名却有 **throws**，所以它可以处理受检异常；

所以归纳起来看主要有这几处不同点：

![img](https://pics7.baidu.com/feed/e1fe9925bc315c6021366d8fec7739154854777f.jpeg?token=2700db476839b8c411691193240956f1)

整体差别虽然不大，但是这点差别，却具有重大意义

返回值和处理异常很好理解，另外，在实际工作中，我们通常要使用线程池来管理线程（**原因已经在 为什么要使用线程池? 中明确说明**），所以我们就来看看 ExecutorService 中是如何使用二者的

**ExecutorService**

先来看一下 ExecutorService 类图

![img](https://pics4.baidu.com/feed/91529822720e0cf34ff52ae86f800019bf09aaf4.jpeg?token=6158be8292a14f30bc9ce17eb89b6d95)

我将上图标记的方法单独放在此处

![img](https://pics3.baidu.com/feed/8718367adab44aedd2144b13d2da7507a08bfb39.jpeg?token=95a176f72890bbec86b4200a476662fe)

可以看到，使用ExecutorService 的 execute() 方法依旧得不到返回值，而 submit() 方法清一色的返回 Future 类型的返回值

细心的朋友可能已经发现， submit() 方法已经在 CountDownLatch 和 CyclicBarrier 傻傻的分不清楚？ 文章中多次使用了，只不过我们没有获取其返回值罢了，那么

Future 到底是什么呢？怎么通过它获取返回值呢？我们带着这些疑问一点点来看

**Future**

Future 又是一个接口，里面只有五个方法：

![img](https://pics0.baidu.com/feed/d0c8a786c9177f3e6fd0a9001209c9c19e3d5627.jpeg?token=657d7a467124292f04a7ac1afb65ce73)

从方法名称上相信你已经能看出这些方法的作用

![img](https://pics3.baidu.com/feed/342ac65c103853438e6f2b99f1d54278cb808837.jpeg?token=2ecbfaf565a7b2e7bccfb8b2b3aac23d)

铺垫了这么多，看到这你也许有些乱了，咱们赶紧看一个例子，演示一下几个方法的作用

![img](https://pics6.baidu.com/feed/c8ea15ce36d3d539e67c8cad58411b56342ab06c.jpeg?token=b10ff28fdbd4cda65f40dc76c7a56b0c)

程序运行结果如下：

![img](https://pics5.baidu.com/feed/8644ebf81a4c510fdd7d23c8039fd72bd62aa5f0.jpeg?token=f6c91cd0781600a4222ca484c731ad69)

如果你运行上述示例代码，主线程调用 future.get() 方法会阻塞自己，直到子任务完成。我们也可以使用 Future 方法提供的 isDone 方法，它可以用来检查 task 是否已经完成了，我们将上面程序做点小修改：

![img](https://pics3.baidu.com/feed/cc11728b4710b91272967f0da33b0e059345223f.jpeg?token=e162e0c65f4e513e8f7c064121ec1207)

来看运行结果：

![img](https://pics4.baidu.com/feed/aec379310a55b319bcccf82b206f7020cefc1787.jpeg?token=785fb43cd3eaa8483e1581aade8624bd)

如果子程序运行时间过长，或者其他原因，我们想 cancel 子程序的运行，则我们可以使用 Future 提供的 cancel 方法，继续对程序做一些修改

![img](https://pics1.baidu.com/feed/472309f790529822c8fd3cb6b50c89cd0846d4c2.jpeg?token=110e5926a5d469b661e551beadfcde97)

来看运行结果：

![img](https://pics6.baidu.com/feed/dc54564e9258d10994b8b6c7b39e3eb96d814dbc.jpeg?token=b037343d56dae18acdeb53edc746f3d6)

为什么调用 cancel 方法程序会出现 CancellationException 呢？ 是因为调用 get() 方法时，明确说明了：

调用 get() 方法时，如果计算结果被取消了，则抛出 CancellationException （具体原因，你会在下面的源码分析中看到）

![img](https://pics0.baidu.com/feed/2fdda3cc7cd98d106fad812943f94a087aec90fe.jpeg?token=adc929ef713a62ff0e4087979eea7dc8)

有异常不处理是非常不专业的，所以我们需要进一步修改程序，以更友好的方式处理异常

![img](https://pics2.baidu.com/feed/a1ec08fa513d2697c11a7b78373d40fd4216d813.jpeg?token=88cc940403b783a4ef57767587669f31)

查看程序运行结果：

![img](https://pics1.baidu.com/feed/c75c10385343fbf21f8df193d2b8388664388f2e.jpeg?token=f7589e9cbc3b0a55713c02f852bbc5c5)

相信到这里你已经对 Future 的几个方法有了基本的使用印象，但 Future 是接口，其实使用 ExecutorService.submit() 方法返回的一直都是 Future 的实现类 FutureTask

![img](https://pics6.baidu.com/feed/8718367adab44aed81c58f39d9da7507a38bfbb5.jpeg?token=e42846dd4df2a96257a209ac8ae3b7e9)

接下来我们就进入这个核心实现类一探究竟

**FutureTask**

同样先来看类结构

![img](https://pics3.baidu.com/feed/78310a55b319ebc4aca9ce0ce0e03dfa1c17169a.jpeg?token=62d3380d9462434f88a02b72d94ef693)

![img](https://pics3.baidu.com/feed/1b4c510fd9f9d72ae0075e37b6ecda32359bbb82.jpeg?token=a4378c5322f58df7af4f552bea318ea4)

很神奇的一个接口，FutureTask 实现了 RunnableFuture 接口，而 RunnableFuture 接口又分别实现了 Runnable 和 Future 接口，所以可以推断出 FutureTask 具有这两种接口的特性：

有 Runnable 特性，所以可以用在 ExecutorService 中配合线程池使用有 Future 特性，所以可以从中获取到执行结果**FutureTask源码分析**

如果你完整的看过 AQS 相关分析的文章，你也许会发现，阅读 Java 并发工具类源码，我们无非就是要关注以下这三点：

![img](https://pics5.baidu.com/feed/14ce36d3d539b600d51c20148996c72cc45cb794.jpeg?token=5e902a416b7a4beeab182c5002a018e8)

脑海中牢记这三点，咱们开始看 FutureTask 源码，看一下它是如何围绕这三点实现相应的逻辑的

文章开头已经提到，实现 Runnable 接口形式创建的线程并不能获取到返回值，而实现 Callable 的才可以，所以 FutureTask 想要获取返回值，必定是和 Callable 有联系的，这个推断一点都没错，从构造方法中就可以看出来：

![img](https://pics4.baidu.com/feed/58ee3d6d55fbb2fb4e28b8122d8cd2a24723dcf8.jpeg?token=709844ea2876bbd452c091892199e242)

即便在 FutureTask 构造方法中传入的是 Runnable 形式的线程，该构造方法也会通过 Executors.callable 工厂方法将其转换为 Callable 类型：

![img](https://pics6.baidu.com/feed/86d6277f9e2f070850d253b88be24a9fab01f284.jpeg?token=4c6a9a4e1d23c4f125369086186c13b0)

但是 FutureTask 实现的是 Runnable 接口，也就是只能重写 run() 方法，run() 方法又没有返回值，那问题来了：

FutureTask 是怎样在 run() 方法中获取返回值的？它将返回值放到哪里了？get() 方法又是怎样拿到这个返回值的呢？

我们来看一下 run() 方法（关键代码都已标记注释）

![img](https://pics4.baidu.com/feed/4b90f603738da97738b80230d3970a1f8718e367.jpeg?token=6eb1e6f5de68d43010932fa7f9e628ac)

run() 方法没有返回值，至于 run() 方法是如何将 call() 方法的返回结果和异常都保存起来的呢？其实非常简单, 就是通过 set(result) 保存正常程序运行结果，或通过 setException(ex) 保存程序异常信息

![img](https://pics0.baidu.com/feed/8644ebf81a4c510f0d5ef243029fd72bd52aa51a.jpeg?token=4d0b57a1a8ce8e90c703ea39941e5a39)

setException 和 set 方法非常相似，都是将异常或者结果保存在 Object 类型的 outcome 变量中，outcome 是成员变量，就要考虑线程安全，所以他们要通过 CAS方式设置 outcome 变量的值，既然是在 CAS 成功后 更改 outcome 的值，这也就是 outcome 没有被 volatile 修饰的原因所在。

保存正常结果值（set方法）与保存异常结果值（setException方法）两个方法代码逻辑，唯一的不同就是 CAS 传入的 state 不同。我们上面提到，state 多数用于控制代码逻辑，FutureTask 也是这样，所以要搞清代码逻辑，我们需要先对 state 的状态变化有所了解

![img](https://pics6.baidu.com/feed/730e0cf3d7ca7bcb709886aadccf9965f424a8a5.jpeg?token=cd1a60cd8359db546d1a0187eaf86f1b)

7种状态，千万别慌，整个状态流转其实只有四种线路

FutureTask 对象被创建出来，state 的状态就是 NEW 状态，从上面的构造函数中你应该已经发现了，四个最终状态 NORMAL ，EXCEPTIONAL ， CANCELLED ， INTERRUPTED 也都很好理解，两个中间状态稍稍有点让人困惑:

COMPLETING: outcome 正在被set 值的时候INTERRUPTING：通过 cancel(true) 方法正在中断线程的时候总的来说，这两个中间状态都表示一种瞬时状态，我们将几种状态图形化展示一下：

![img](https://pics7.baidu.com/feed/6a63f6246b600c33e8706e0d798aa309d8f9a133.jpeg?token=2d2366a33ee1e47398b03510dc1deef0)

我们知道了 run() 方法是如何保存结果的，以及知道了将正常结果/异常结果保存到了 outcome 变量里，那就需要看一下 FutureTask 是如何通过 get() 方法获取结果的：

![img](https://pics4.baidu.com/feed/a686c9177f3e6709a5478d9059016d3bf9dc5563.jpeg?token=35926d941da77a9659b3f0af3e56ff61)

awaitDone 方法是 FutureTask 最核心的一个方法

![img](https://pics4.baidu.com/feed/d6ca7bcb0a46f21f1a222a2a9be299660d33ae86.jpeg?token=30ba33ad8f272614336e9b5c422faf9e)

总的来说，进入这个方法，通常会经历三轮循环

第一轮for循环，执行的逻辑是 q == null, 这时候会新建一个节点 q, 第一轮循环结束。第二轮for循环，执行的逻辑是 !queue，这个时候会把第一轮循环中生成的节点的 next 指针指向waiters，然后CAS的把节点q 替换waiters, 也就是把新生成的节点添加到waiters 中的首节点。如果替换成功，queued=true。第二轮循环结束。第三轮for循环，进行阻塞等待。要么阻塞特定时间，要么一直阻塞知道被其他线程唤醒。对于第二轮循环，大家可能稍稍有点迷糊，我们前面说过，有阻塞，就会排队，有排队自然就有队列，FutureTask 内部同样维护了一个队列

![img](https://pics3.baidu.com/feed/d53f8794a4c27d1e5471f69d79135f68dcc43865.jpeg?token=22e773c6036229618a350848dedbd5f6)

说是等待队列，其实就是一个 Treiber 类型 stack，既然是 stack， 那就像手枪的弹夹一样（脑补一下子弹放入弹夹的情形），后进先出，所以刚刚说的第二轮循环，会把新生成的节点添加到 waiters stack 的首节点

如果程序运行正常，通常调用 get() 方法，会将当前线程挂起，那谁来唤醒呢？自然是 run() 方法运行完会唤醒，设置返回结果（set方法）/异常的方法(setException方法) 两个方法中都会调用 finishCompletion 方法，该方法就会唤醒等待队列中的线程

![img](https://pics5.baidu.com/feed/0b55b319ebc4b745801a66b2ab3aec11888215d1.jpeg?token=849aa310885aeedd8c6275c50dd7e5dd)

将一个任务的状态设置成终止态只有三种方法：

setsetExceptioncancel前两种方法已经分析完，接下来我们就看一下 cancel 方法查看 Future cancel()，该方法注释上明确说明三种 cancel 操作一定失败的情形

任务已经执行完成了任务已经被取消过了任务因为某种原因不能被取消其它情况下，cancel操作将返回true。值得注意的是，cancel操作返回 true 并不代表任务真的就是被取消, **这取决于发动cancel状态时，任务所处的状态**

如果发起cancel时任务还没有开始运行，则随后任务就不会被执行；如果发起cancel时任务已经在运行了，则这时就需要看 mayInterruptIfRunning 参数了：如果mayInterruptIfRunning 为true, 则当前在执行的任务会被中断如果mayInterruptIfRunning 为false, 则可以允许正在执行的任务继续运行，直到它执行完有了这些铺垫，看一下 cancel 代码的逻辑就秒懂了

![img](https://pics2.baidu.com/feed/6d81800a19d8bc3ea8a37309e04d5418aad34592.jpeg?token=12395f7ea93fdc49b534a18171af0afb)

核心方法终于分析完了，到这咱们喝口茶休息一下吧

我是想说，使用 FutureTask 来演练烧水泡茶经典程序

![img](https://pics0.baidu.com/feed/a71ea8d3fd1f41347c6a45a447d967ccd3c85edc.jpeg?token=01d533a31d4f063e435aa5450eae5ff9)

如上图：

洗水壶 1 分钟烧开水 15 分钟洗茶壶 1 分钟洗茶杯 1 分钟拿茶叶 2 分钟最终泡茶

让我心算一下，如果串行总共需要 20 分钟，但很显然在烧开水期间，我们可以洗茶壶/洗茶杯/拿茶叶

![img](https://pics2.baidu.com/feed/f2deb48f8c5494eeed5618b44e3312f898257e77.jpeg?token=3ceed3faf0b366f23f2bf864c00ef05b)

这样总共需要 16 分钟，节约了 4分钟时间，烧水泡茶尚且如此，在现在高并发的时代，4分钟可以做的事太多了，学会使用 Future 优化程序是必然（**其实优化程序就是寻找关键路径，关键路径找到了，非关键路径的任务通常就可以和关键路径的内容并行执行了**）

![img](https://pics5.baidu.com/feed/b21bb051f819861868db8c1e2c2bdc7589d4e6dd.jpeg?token=7f362d1682f595e2746c52afdbec5d94)

上面的程序是主线程等待两个 FutureTask 的执行结果，线程1 烧开水时间更长，线程1希望在水烧开的那一刹那就可以拿到茶叶直接泡茶，怎么半呢？

![img](https://pics5.baidu.com/feed/3ac79f3df8dcd10061d541a4184db516b8122f33.jpeg?token=5ef6b65bd9a32e57bb6a1d0244af8650)

那只需要在线程 1 的FutureTask 中获取 线程 2 FutureTask 的返回结果就可以了，我们稍稍修改一下程序：

![img](https://pics2.baidu.com/feed/d1160924ab18972b2066b8b78b0b898f9f510ab8.jpeg?token=c114a4bda0339742b655258f08cead7f)

来看程序运行结果：

![img](https://pics3.baidu.com/feed/4a36acaf2edda3ccde0f0f2f632fcb07203f9227.jpeg?token=38e05122b1a685e1e4bcca56333e1f73)

知道这个变化后我们再回头看 ExecutorService 的三个 submit 方法：

![img](https://pics1.baidu.com/feed/d52a2834349b033bf0299f727708c4d5d739bdc6.jpeg?token=6d0fd691d0b6a1a2dab0a7052bcda8ee)

第一种方法，逐层代码查看到这里：

![img](https://pics4.baidu.com/feed/962bd40735fae6cd2ce4a95a6c75fd2243a70f81.jpeg?token=45376915f6758532f0dd7986869772a5)

你会发现，和我们改造烧水泡茶的程序思维是相似的，可以传进去一个 result，result 相当于主线程和子线程之间的桥梁，通过它主子线程可以共享数据

第二个方法参数是 Runnable 类型参数，即便调用 get() 方法也是返回 null，所以仅是可以用来断言任务已经结束了，类似 Thread.join()

第三个方法参数是 Callable 类型参数，通过get() 方法可以明确获取 call() 方法的返回值

到这里，关于 Future 的整块讲解就结束了，还是需要简单消化一下的

**总结**

如果熟悉 Javascript 的朋友，Future 的特性和 Javascript 的 Promise 是类似的，私下开玩笑通常将其比喻成**男朋友的承诺**

回归到Java，我们从 JDK 的演变历史，谈及 Callable 的诞生，它弥补了 Runnable 没有返回值的空缺，通过简单的 demo 了解 Callable 与 Future 的使用。 FutureTask 又是 Future接口的核心实现类，通过阅读源码了解了整个实现逻辑，最后结合FutureTask 和线程池演示烧水泡茶程序，相信到这里，你已经可以轻松获取线程结果了

烧水泡茶是非常简单的，如果更复杂业务逻辑，以这种方式使用 Future 必定会带来很大的会乱（程序结束没办法主动通知，Future 的链接和整合都需要手动操作）为了解决这个短板，没错，又是那个男人 Doug Lea, CompletableFuture 工具类在 Java1.8 的版本出现了，搭配 Lambda 的使用，让我们编写异步程序也像写串行代码那样简单，纵享丝滑

![img](https://pics3.baidu.com/feed/0d338744ebf81a4cc868b9f5b6ec925f242da685.jpeg?token=f7807f9fe2aa127c039dfe1c4a7ae58b)