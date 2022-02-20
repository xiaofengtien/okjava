# 概述

ReentrantLock常常对比着synchronized来分析，我们先对比着来看然后再一点一点分析。

（1）synchronized是独占锁，加锁和解锁的过程自动进行，易于操作，但不够灵活。ReentrantLock也是独占锁，加锁和解锁的过程需要手动进行，不易操作，但非常灵活。

（2）synchronized可重入，因为加锁和解锁自动进行，不必担心最后是否释放锁；ReentrantLock也可重入，但加锁和解锁需要手动进行，且次数需一样，否则其他线程无法获得锁。

（3）synchronized不可响应中断，一个线程获取不到锁就一直等着；ReentrantLock可以相应中断。

ReentrantLock好像比synchronized关键字没好太多，我们再去看看synchronized所没有的，一个最主要的就是ReentrantLock还可以实现公平锁机制。什么叫公平锁呢？也就是在锁上等待时间最长的线程将获得锁的使用权。通俗的理解就是谁排队时间最长谁先执行获取锁

# 非公平锁的加锁流程

​	state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

　　任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

​	当我们调用ReentrantLock的lock方法的时候，实际上是调用了NonfairSync的lock方法，这个方法先用CAS操作，去尝试抢占该锁。如果成功，就把当前线程设置在这个锁上，表示抢占成功。如果失败，则调用acquire模板方法，等待抢占。代码如下：

```java
static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
 
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
}
```

## nonfairTryAcquire

NonfairSync这个类中其实就是使用了nonfairTryAcquire，具体实现原理是先比较当前锁的状态是否是0，如果是0，则尝试去原子抢占这个锁（设置状态为1，然后把当前线程设置成独占线程），如果当前锁的状态不是0，就去比较当前线程和占用锁的线程是不是一个线程，如果是，会去增加状态变量的值，从这里看出可重入锁之所以可重入，就是同一个线程可以反复使用它占用的锁。如果以上两种情况都不通过，则返回失败false。代码如下

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    //（1）获取当前线程
    final Thread current = Thread.currentThread();
    //（2）获得当前同步状态state
    int c = getState();
    //（3）如果state==0，表示没有线程获取
    if (c == 0) {
        //（3-1）那么就尝试以CAS的方式更新state的值
        if (compareAndSetState(0, acquires)) {
            //（3-2）如果更新成功，就设置当前独占模式下同步状态的持有者为当前线程
            setExclusiveOwnerThread(current);
            //（3-3）获得成功之后，返回true
            return true;
        }
    }
    //（4）这里是重入锁的逻辑
    else if (current == getExclusiveOwnerThread()) {
        //（4-1）判断当前占有state的线程就是当前来再次获取state的线程之后，就计算重入后的state
        int nextc = c + acquires;
        //（4-2）这里是风险处理
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        //（4-3）通过setState无条件的设置state的值，（因为这里也只有一个线程操作state的值，即
        //已经获取到的线程，所以没有进行CAS操作）
        setState(nextc);
        return true;
    }
    //（5）没有获得state，也不是重入，就返回false
    return false;
}
```

# 公平锁和非公平锁的区别

公平锁和非公平锁，在CHL队列抢占模式上都是一致的，也就是在进入acquireQueued这个方法之后都一样，它们的区别在初次抢占上有区别，也就是tryAcquire上的区别，下面是两者内部调用关系的简图：

```powershell
NonfairSync
lock —> compareAndSetState
                | —> setExclusiveOwnerThread
      —> accquire
             | —> tryAcquire
                           |—>nonfairTryAcquire
                |—> acquireQueued

FairSync
lock —> acquire
               | —> tryAcquire
                           |—>!hasQueuePredecessors
                           |—>compareAndSetState
                           |—>setExclusiveOwnerThread
               |—> acquireQueued
```

真正的区别就是公平锁多了hasQueuePredecessors这个方法，这个方法用于判断CHL队列中是否有节点，对于公平锁，如果CHL队列有节点，则新进入竞争的线程一定要在CHL上排队，而非公平锁则是无视CHL队列中的节点，直接进行竞争抢占，这就有可能导致CHL队列上的节点永远获取不到锁，这就是非公平锁之所以不公平的原因

**非公平锁和公平锁的两处不同：**

1. 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
2. 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。

公平锁和非公平锁就这两点区别，如果这两次 CAS 都不成功，那么后面非公平锁和公平锁是一样的，都要进入到阻塞队列等待唤醒。

相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致在阻塞队列中的线程长期处于饥饿状态。