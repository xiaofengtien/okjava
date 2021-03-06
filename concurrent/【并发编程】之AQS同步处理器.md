# 概述

​		 AbstractQueuedSynchronizer抽象队列同步器简称AQS，它是实现同步器的基础组件，它维护了一个volatile int state（代表共享资源）和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）PS:AQS是基于模板模式对外提供acquire*等方法由自定义同步器实现，内部通过维护CLH队列实现线程同步和通讯。

1、AQS通过维护内置的FIFO双向队列（CLH）来完成线程的排队工作，内部通过节点head和tail分别记录队首和队尾元素，元素节点类型为Node类型

```java
/*等待队列的队首节点(懒加载，这里体现为竞争失败的情况下，加入同步队列的线程执行到enq方法的时候会创
建一个Head结点)。该节点只能被setHead方法修改。并且节点的waitStatus不能为CANCELLED*/
private transient volatile Node head;
/**等待队列的尾节点，也是懒加载的。（enq方法）。只在加入新的阻塞节点的情况下通过getAndset修改*/
private transient volatile Node tail;
```

2、Node中的Thread用来存放进入AQS队列的线程引用，Node内部的SHARED表示线程是因为获取共享资源失败被阻塞添加到队列中的；Node中的EXCLUSIVE表示线程因为获取独占资源失败被阻塞添加到队列中的，waitStatus表示当前线程的等待状态：

waitStatus:

​	a.CANCELLED=1:表示线程因为中断或者等待超时，需要从等待队列中取消等待，

​	b.SIGNEL=-1：当前线程独占锁，队列的head（仅仅代表头节点，里面没有存放线程引用）的后继节点node1处于等待状态，如果已占有锁的线程释放锁或者被CANCEL之后就会通知该节点node1去获取锁执行

​	c.CONDITION=-2:表示节点在等待队列中，这里指的是等待在某个lock的condition上，当持有锁的线程调用了condition的signal（）方法后，节点会从该condition的等待队列转移到lock的同步队列中去竞争lock。

​	d.PROPAGTE=-3:表示下一次共享状态获取将会传递给后继节点获取这个共享同步状态

3、AQS中维护一个单一的volatile修饰的状态信息state（AQS通过Unsafe的相关方法，以原子性的方式由线程去获取这个state，AQS提供了getSate、setState、compareAndSetState函数修改值，实际上调用的是Unsafe的compareAndSwapInt方法）

```java
//这就是我们刚刚说到的head结点，懒加载的（只有竞争失败需要构建同步队列的时候，才会创建这个head），如果头节点存在，它的waitStatus不能为CANCELLED
private transient volatile Node head;
//当前同步队列尾节点的引用，也是懒加载的，只有调用enq方法的时候会添加一个新的wait node
private transient volatile Node tail;
//AQS核心：同步状态
private volatile int state;
protected final int getState() {
    return state;
}
protected final void setState(int newState) {
    state = newState;
}
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

4、AQS基于模板方法模式，使用时候需要继承同步器并重写指定方法，并且通常将子类推荐为定义同步组件的静态内部类。

```java
isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
//独占式的获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态
protected boolean tryAcquire(int arg) {	throw new UnsupportedOperationException();}
//独占式的释放同步状态，等待获取同步状态的线程可以有机会获取同步状态
protected boolean tryRelease(int arg) {	throw new UnsupportedOperationException();}
//共享式的获取同步状态
protected int tryAcquireShared(int arg) { throw new UnsupportedOperationException();}
//尝试将状态设置为以共享模式释放同步状态。 该方法总是由执行释放的线程调用。 
protected int tryReleaseShared(int arg) { throw new UnsupportedOperationException(); }
//当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占
protected int isHeldExclusively(int arg) {	throw new UnsupportedOperationException();}
```

5、AQS的内部类**ConditionObject**是通过结合锁实现线程同步，ConditionObject可以直接访问AQS的变量(state、queue)，ConditionObject是个条件变量 ，每个ConditionObject对应一个队列用来存放线程调用condition条件变量的await方法之后被阻塞的线程

# 独占模式-入列

​		AQS会将所有的请求线程组成一个CLH队列，当一个线程执行完毕（unlock）会激活自己的后继节点，但正在执行的线程并不在队列中，而那些等待执行的线程全部处于阻塞状态（park）

## acquire(int)

​	此方法是独占模式下线程获取共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。这也正是lock()的语义，当然不仅仅只限于lock()。获取到资源后，线程就可以去执行其临界区代码了。下面是acquire()的源码：

```java
 public final void acquire(int arg) {
     if (!tryAcquire(arg) &&
         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
 }
```

　函数流程如下：

1.tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2.addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3.acquireQueued()使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4.如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

##  tryAcquire(int)

​	此方法尝试去获取独占资源。如果获取成功，则直接返回true，否则直接返回false。这也正是tryLock()的语义，还是那句话，当然不仅仅只限于tryLock()。如下是tryAcquire()的源码：

```java 
protected boolean tryAcquire(int arg) {
       throw new UnsupportedOperationException();
}
```

这里之所以没有定义成abstract，是因为独占模式下只用实现tryAcquire-tryRelease，而共享模式下只用实现tryAcquireShared-tryReleaseShared。如果都定义成abstract，那么每个模式也要去实现另一模式下的接口。说到底，Doug Lea还是站在咱们开发者的角度，尽量减少不必要的工作量。

## addWaiter(Node)

​	此方法用于将当前线程加入到等待队列的队尾，并返回当前线程所在的结点。还是上源码吧：

```java
private Node addWaiter(Node mode) {
    //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
    Node node = new Node(Thread.currentThread(), mode);
    
    //尝试快速方式直接放到队尾。将构造后的node结点的前驱结点设置为tail
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //以CAS的方式设置当前的node结点为tail结点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    //上一步失败则通过enq入队。
    enq(node);
    return node;
}
```

## enq(Node)

​	此方法用于将node加入队尾。源码如下：

```java
 private Node enq(final Node node) {
     //CAS"自旋"，直到成功加入队尾
     for (;;) {
         Node t = tail;
         if (t == null) { // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。如果tail为null，表示当前同步队列为null，就必须初始化这个同步队列的head和tail（建立一个哨兵结点）
             if (compareAndSetHead(new Node()))
                 //初始情况下，多个线程竞争失败，在检查的时候都发现没有哨兵结点，所以需要CAS的设置哨兵结点
                 tail = head;
         } else {//正常流程，放入队尾
             //直接将当前结点的前驱结点设置为tail结点
             node.prev = t;
             //前驱结点设置完毕之后，还需要以CAS的方式将自己设置为tail结点，如果设置失败,就会重新进入循环判断一遍
             if (compareAndSetTail(t, node)) {
                 t.next = node;
                 return t;
             }
         }
     }
 }
```

## acquireQueued(Node, int)

​	通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。**进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源**，如果当前节点是头节点的下个节点，并且通过tryAcquire获取到了锁，说明头节点已经释放了锁，当前线程是被头节点那个线程唤醒的，这时候就可以将当前节点设置成头节点，并且将failed标记设置成false，然后返回。至于上一个节点，它的next变量被设置为null，在下次GC的时候会清理掉

```java
final boolean acquireQueued(final Node node, int arg) {
     boolean failed = true;//标记是否成功拿到资源
     try {
         boolean interrupted = false;//标记等待过程中是否被中断过
         
         //又是一个“自旋”！
         for (;;) {
             final Node p = node.predecessor();//拿到前驱
             //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
             if (p == head && tryAcquire(arg)) {
                 setHead(node);//拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                 p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                 failed = false;
                 return interrupted;//返回等待过程中是否被中断过
             }
             
             //如果自己可以休息了，就进入waiting状态，直到被unpark()
             if (shouldParkAfterFailedAcquire(p, node) &&
                 parkAndCheckInterrupt())
                 interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
         }
     } finally {
         if (failed)
             cancelAcquire(node);
     }
 }
```

到这里了，先看看shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()具体干些什么。

##  shouldParkAfterFailedAcquire(Node, Node)

​	此方法主要用于检查状态，

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
     int ws = pred.waitStatus;//拿到前驱的状态
     if (ws == Node.SIGNAL)
         //如果前驱结点的waitStatus为SINGNAL，就直接返回true，前驱结点的状态为SIGNAL，那么该结点就能够安全的调用park方法阻塞自己了。
         return true;
     if (ws > 0) {
         /*
          * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
          *这里就是将所有的前驱结点状态为CANCELLED的都移除 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被(GC回收)！
          */
         do {
             node.prev = pred = pred.prev;
         } while (pred.waitStatus > 0);
         pred.next = node;
     } else {
          //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
         compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
     }
     return false;
 }
```

如果尝试获取锁失败，就会进入shouldParkAfterFailedAcquire方法，会判断当前线程是否挂起，如果前一个节点已经是SIGNAL状态，则当前线程需要挂起。如果前一个节点是取消状态，则需要将取消节点从队列移除。如果前一个节点状态是其他状态，则尝试设置成SIGNAL状态，并返回不需要挂起，从而进行第二次抢占。完成上面的事后进入挂起阶段

## parkAndCheckInterrupt()

​	如果线程找好安全休息点后，那就可以安心去休息了。此方法就是让线程去休息，真正进入等待状态。

```java
 private final boolean parkAndCheckInterrupt() {
     LockSupport.park(this);//调用park()使线程进入waiting状态
     return Thread.interrupted();//如果被唤醒，查看自己是不是被中断的。
 }
```

park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()，需要注意的是，Thread.interrupted()会清除当前线程的中断标记位

# 独占模式-出列

## release(int)

​	此方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。这也正是unlock()的语义，当然不仅仅只限于unlock()。下面是release()的源码：

```java
 public final boolean release(int arg) {
     if (tryRelease(arg)) {
         Node h = head;//找到头结点
         if (h != null && h.waitStatus != 0)
             unparkSuccessor(h);//唤醒等待队列里的下一个线程
         return true;
     }
     return false;
 }
```

它调用tryRelease()来释放资源。有一点需要注意的是，**它是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自定义同步器在设计tryRelease()的时候要明确这一点！！**

## tryRelease(int)

​	此方法尝试去释放指定量的资源。下面是tryRelease()的源码：

```java
 protected boolean tryRelease(int arg) {
     throw new UnsupportedOperationException();
 }
```

跟tryAcquire()一样，这个方法是需要独占模式的自定义同步器去实现的。正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，**release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！**所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false

## unparkSuccessor(Node)

​	此方法用于唤醒等待队列中下一个线程。下面是源码：

```java
 private void unparkSuccessor(Node node) {
     //这里，node一般为当前线程所在的结点。
     int ws = node.waitStatus;
     if (ws < 0)//置零当前线程所在的结点状态，允许失败。
         compareAndSetWaitStatus(node, ws, 0);
 
     Node s = node.next;//找到下一个需要唤醒的结点s
     if (s == null || s.waitStatus > 0) {//如果为空或已取消
         s = null;
         for (Node t = tail; t != null && t != node; t = t.prev)
             if (t.waitStatus <= 0)//从这里可以看出，<=0的结点，都是还有效的结点。
                 s = t;
     }
     if (s != null)
         LockSupport.unpark(s.thread);//唤醒
 }
```

这个函数并不复杂。一句话概括：**用unpark()唤醒等待队列中最前边的那个未放弃线程**，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了

release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源

# 共享模式-入列

## acquireShared(int)

​	此方法是共享模式下线程获取共享资源的顶层入口。它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。下面是acquireShared()的源码

```java

public final void acquireShared(int arg) {
     if (tryAcquireShared(arg) < 0)
         doAcquireShared(arg);
 }
```

　　这里tryAcquireShared()依然需要自定义同步器去实现。但是AQS已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。所以这里acquireShared()的流程就是：

1、tryAcquireShared()尝试获取资源，成功则直接返回；
2、失败则通过doAcquireShared()进入等待队列，直到获取到资源为止才返回

## doAcquireShared(int)

​	此方法用于将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。下面是doAcquireShared()的源码：

```java
private void doAcquireShared(int arg) {
     final Node node = addWaiter(Node.SHARED);//加入队列尾部
     boolean failed = true;//是否成功标志
     try {
         boolean interrupted = false;//等待过程中是否被中断过的标志
         for (;;) {
             final Node p = node.predecessor();//前驱
             if (p == head) {//如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
                 int r = tryAcquireShared(arg);//尝试获取资源
                 if (r >= 0) {//成功
                     setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以再唤醒之后的线程
                     p.next = null; // help GC
                     if (interrupted)//如果等待过程中被打断过，此时将中断补上。
                         selfInterrupt();
                     failed = false;
                     return;
                 }
             }
             
             //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
             if (shouldParkAfterFailedAcquire(p, node) &&
                 parkAndCheckInterrupt())
                 interrupted = true;
         }
     } finally {
         if (failed)
             cancelAcquire(node);
     }
 }
```

跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。当然，这并不是问题，只是AQS保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）

## setHeadAndPropagate(Node, int)

此方法在setHead()的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点，毕竟是共享模式！

```java
private void setHeadAndPropagate(Node node, int propagate) {
     Node h = head; 
     setHead(node);//head指向自己
      //如果还有剩余量，继续唤醒下一个邻居线程
     if (propagate > 0 || h == null || h.waitStatus < 0) {
         Node s = node.next;
         if (s == null || s.isShared())
             doReleaseShared();
     }
 }
```

　　其实跟acquire()的流程大同小异，只不过多了个**自己拿到资源后，还会去唤醒后继队友的操作**

# 共享模式-出列

## releaseShared()

​	此方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。下面是releaseShared()的源码：

```java
 public final boolean releaseShared(int arg) {
     if (tryReleaseShared(arg)) {//尝试释放资源
         doReleaseShared();//唤醒后继结点
         return true;
     }
     return false;
 }
```

此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以自定义同步器可以根据需要决定tryReleaseShared()的返回值

## doReleaseShared()

此方法主要用于唤醒后继。下面是它的源码：

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);//唤醒后继
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        if (h == head)// head发生变化
            break;
    }
}
```

# Mutex（互斥锁）

Mutex是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。下边是Mutex的核心源码：

```java
class Mutex implements Lock, java.io.Serializable {
    // 自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 判断是否锁定状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 尝试获取资源，立即返回。成功则返回true，否则false。
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // 这里限定只能为1个量
            if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
                setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
                return true;
            }
            return false;
        }

        // 尝试释放资源，立即返回。成功则为true，否则false。
        protected boolean tryRelease(int releases) {
            assert releases == 1; // 限定为1个量
            if (getState() == 0)//既然来释放，那肯定就是已占有状态了。只是为了保险，多层判断！
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);//释放资源，放弃占有状态
            return true;
        }
    }

    // 真正同步类的实现都依赖继承于AQS的自定义同步器！
    private final Sync sync = new Sync();

    //lock<-->acquire。两者语义一样：获取资源，即便等待，直到成功才返回。
    public void lock() {
        sync.acquire(1);
    }

    //tryLock<-->tryAcquire。两者语义一样：尝试获取资源，要求立即返回。成功则为true，失败则为false。
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    //unlock<-->release。两者语文一样：释放资源。
    public void unlock() {
        sync.release(1);
    }

    //锁是否占有状态
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
}
```

同步类在实现时一般都将自定义同步器（sync）定义为内部类，供自己使用；而同步类自己（Mutex）则实现某个接口，对外服务。当然，接口的实现要直接依赖sync，它们在语义上也存在某种对应关系！！而sync只用实现资源state的获取-释放方式tryAcquire-tryRelelase，至于线程的排队、等待、唤醒等，上层的AQS都已经实现好了，我们不用关心。