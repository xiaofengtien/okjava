# CLH(Craig, Landin, andHagersten)根据三个人的名字命名的队列

# SMP 和 NUMA 简要介绍

- **SMP** (Symmetric MultiProcessing) 对称多处理是一种包括软硬件的多核计算机架构，会有两个或以上的相同的核心共享一块主存，这些核心在操作系统中地位相同，可以访问所有 I/O 设备。它的优点是内存等一些组件在核心之间是共享的，一致性可以保证，但也正因为内存一致性和共享对象在扩展性上就受到限制了 。

![img](https://imgconvert.csdnimg.cn/aHR0cDovL21lbmVsdGFybWEtcGljdHVyZXMubm9zLWVhc3RjaGluYTEuMTI2Lm5ldC9jb25jdXJyZW50L0NMSC9TTVAucG5n?x-oss-process=image/format,png)

​																								图1 SMP架构

-  **NUMA** (Non-uniform memory access) 非一致存储访问也是一种在多处理任务中使用的计算机存储设计，每个核心有自己对应的本地内存，各核心之间通过互联进行相互访问。在这种架构下，对内存的访问时间取决于内存地址与具体核心之间的相对地址。在 NUMA 中，核心访问自己的本地内存比访问非本地内存（另一个核心的本地内存或者核心之间的共享内存）要快。NUMA 的优势局限于一些特定任务，尤其是对于那些数据经常与特定任务或者用户具有强关联关系的服务器。解决了SMP的扩展问题，但当核心数量很多时，核心访问非本地内存开销很大，性能增长会减慢。 

![img](https://imgconvert.csdnimg.cn/aHR0cDovL21lbmVsdGFybWEtcGljdHVyZXMubm9zLWVhc3RjaGluYTEuMTI2Lm5ldC9jb25jdXJyZW50L0NMSC9OVU1BLnBuZw?x-oss-process=image/format,png)

​																							图2 NUMA架构

 

# CLH 队列锁

## 简介

在共享内存多处理器环境中，维护共享数据结构的逻辑一致性是一个普遍问题。将这些数据结构用锁保护起来是维持这种一致性的标准技术。需要访问数据的进程（下文中进程，线程，process 都可以看做一个概念，并发运行的程序单位）必须先获取这个数据对应的锁。获取锁之后，进程就独占了对这个数据的访问权知道进行将锁释放。其对锁进行请求的进程都必须进行等待。在持有锁的进程释放锁之后，等待进程的其中一个会获取这把锁，同时其他进程接着等待。

等待进程的等待方式也分两种：被动等待（让出CPU）或者主动等待（自旋）。被动等待就是，进程注册对锁的请求然后阻塞，以便在它等待的时候其他进程可以利用处理器。当锁被释放时，已注册的进程中的其中一个会获取锁。于是被选中的进程就会被解除阻塞在调度就绪时运行。主动等待就是，最典型的就是进程进入一个不断重复检测锁状态并且/或者尝试获取锁对象的紧凑循环（tight loop）。一旦它获取锁对象，就进入受保护数据运行程序。

Anderson[2] 和 Mellor-Crummey 与 Scott[3]提供了对等待方式优缺点的讨论。直观上感觉自旋就是CPU在空转，肯定比阻塞等待浪费性能，但实际上对于小任务，空转时间很短，锁很快就被释放，与阻塞方式在进程状态管理和切换不可忽略的系统开销相比，自旋的代价比阻塞和恢复进程反而小。CLH 锁就是自旋锁的这种被动方式的实践。

队列自旋锁的一个潜在优势就是等待进程不在同一个内存地址上自旋。对于 NUMA 甚至可以达到每个进程都在处理的核心对应的本地内存上自旋，就这降低了各个核心和内存之间互联互通的负载。尤其是在对于某一时间若干等待进程对锁的高争用情况，这点尤其重要。另外队列自旋锁还可以用 FIFO 队列保证对进程的某种公平性和对避免饥饿的保证。

## CLH 队列锁中的结构（FIFO 队列）

没有特殊情况我们面对的基本上都是 SMP 架构的系统，这里就只分析最基础的对于 FIFO 队列的锁，优先队列锁和对于 NUMA 系统的锁不做解析。

CLH同步队列是一个FIFO双向队列，AQS依赖它来完成同步状态的管理，当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。

在CLH同步队列中，一个节点表示一个线程，它保存着线程的引用（thread）、状态（waitStatus）、前驱节点（prev）、后继节点（next）

![这里写图片描述](https://img-blog.csdn.net/20170307220859962?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 入列

addWaiter(Node node)

enq(Node node)

![这里写图片描述](https://img-blog.csdn.net/20170307220924257?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 出列

CLH同步队列遵循FIFO，首节点的线程释放同步状态后，将会唤醒它的后继节点（next），而后继节点将会在获取同步状态成功时将自己设置为首节点，这个过程非常简单，head执行该节点并断开原首节点的next和当前节点的prev即可，注意在这个过程是不需要使用CAS来保证的，因为只有一个线程能够成功获取到同步状态。过程图如下

![这里写图片描述](https://img-blog.csdn.net/20170307220947523?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)