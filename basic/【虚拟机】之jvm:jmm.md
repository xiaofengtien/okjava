**一 jvm结构
**

jvm的内部结构如下图所示，这张图很清楚形象的描绘了整个JVM的内部结构，以及各个部分之间的交互和作用。

![img](https://img-blog.csdn.net/20170513134212845?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTFpONTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

1 Class Loader（类加载器）就是将Class文件加载到内存，再说的详细一点就是，把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是类加载器的作用。

2 Run Data Area（运行时数据区） 就是我们常说的JVM管理的内存了，也是我们这里主要讨论的部分。运行数据区是整个JVM的重点。我们所有写的程序都被加载到这里，之后才开始运行。这部分也是我们这里将要讨论的重点。

3 Execution engine（执行引擎） 是Java虚拟机最核心的组成部分之一。执行引擎用于执行指令，不同的java虚拟机内部实现中，执行引擎在执行Java代码的时候可能有解释执行（解释器执行）和编译执行（通过即时编译器产生本地代码执行，例如BEA JRockit），也有可能两者兼备。任何JVM specification实现(JDK)的核心都是Execution engine，不同的JDK例如Sun 的JDK 和IBM的JDK好坏主要就取决于他们各自实现的Execution engine的好坏。

4 Native interface 与native libraries交互，是其它编程语言交互的接口。当调用native方法的时候，就进入了一个全新的并且不再受虚拟机限制的世界，所以也很容易出现JVM无法控制的native heap OutOfMemory。

**二 Run Data Area（运行时数据区）**



| 数据                     | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ |
| Program Counter Register | 程序计数器，线程私有、指向下一条要很执行的指令               |
| Java Stack               | Java虚拟机栈，线程私有，生命周期与线程相同。描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口 |
| Native Method Stack      | 为虚拟机使用到的Native 方法服务                              |
| Heap                     | 线程共享，由于现在收集器基本采用的分代收集算法，所以Java堆中还可以细分：新生代和老生代；更细致一点的有Eden空间、From Survivor空间、To Survivor空间等。所有的对象实例以及数组都要在堆上分配，是垃圾收集器管理的主要区域 |
| Method Area              | 方法区，别名叫做非堆(Non-Heap)，线程共享的内存区域。目的是与Java堆区分开来，存储类信息、常量、静态变量、即时编译器编译后的代码。 方法区存放的信息包括： **A 类的基本信息：** 1.每个类的全限定名 2.每个类的直接超类的全限定名(可约束类型转换) 3.该类是类还是接口 4.该类型的访问修饰符 5.直接超接口的全限定名的有序列表 **B 已装载类的详细信息** 1.运行时常量池：在方法区中，每个类型都对应一个常量池，存放该类型所用到的所有常量，常量池中存储了诸如文字字符串、final变量值、类名和方法名常量。它们以数组形式通过索引被访 问，是外部调用与类联系及类型对象化的桥梁。（存的可能是个普通的字符串，然后经过常量池解析，则变成指向某个类的引用） 2.字段信息：字段信息存放类中声明的每一个字段的信息，包括字段的名、类型、修饰符。字段名称指的是类或接口的实例变量或类变量，字段的描述符是一个指示字段的类型的字符串，如private A a=null;则a为字段名，A为描述符，private为修饰符。 3.方法信息：类中声明的每一个方法的信息，包括方法名、返回值类型、参数类型、修饰符、异常、方法的字节码。(在编译的时候，就已经将方法的局部变量、操作数栈大小等确定并存放在字节码中，在装载的时候，随着类一起装入方法区。) 4.静态变量：就是类变量，类的所有实例都共享，我们只需知道，在方法区有个静态区，静态区专门存放静态变量和静态块。 5.到类classloader的引用：到该类的类装载器的引用。 6.到类class 的引用：jvm为每个加载的类型(译者：包括类和接口)都创建一个java.lang.Class的实例。而jvm必须以某种方式把Class的这个实例和存储在方法区中的类型数据联系起来。 |



**三 jmm**

Java内存模型(Java Memory Model，JMM)JMM主要是为了规定了线程和内存之间的一些关系。根据JMM的设计，系统存在一个主内存(Main Memory)，Java中所有变量都储存在主存中，对于所有线程都是共享的。每条线程都有自己的工作内存(Working Memory)，工作内存中保存的是主存中某些变量的拷贝，线程对所有变量的操作都是在工作内存中进行，线程之间无法相互直接访问，变量传递均需要通过主存完成。**
**

![img](https://img-blog.csdn.net/20170513140412095?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTFpONTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**四 jvm和jmm之间的关系**

jmm中的主内存、工作内存与jvm中的Java堆、栈、方法区等并不是同一个层次的内存划分，这两者基本上是没有关系的，如果两者一定要勉强对应起来，那从变量、主内存、工作内存的定义来看，主内存主要对应于Java堆中的对象实例数据部分，而工作内存则对应于虚拟机栈中的部分区域。从更低层次上说，主内存就直接对应于物理硬件的内存，而为了获取更好的运行速度，虚拟机（甚至是硬件系统本身的优化措施）可能会让工作内存优先存储于寄存器和高速缓存中，因为程序运行时主要访问读写的是工作内存

## 内存交互操作

 　内存交互操作有8种，虚拟机实现必须保证每一个操作都是原子的，不可在分的（对于double和long类型的变量来说，load、store、read和write操作在某些平台上允许例外）

- - lock   （锁定）：作用于主内存的变量，把一个变量标识为线程独占状态
  - unlock （解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
  - read  （读取）：作用于主内存变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用
  - load   （载入）：作用于工作内存的变量，它把read操作从主存中变量放入工作内存中
  - use   （使用）：作用于工作内存中的变量，它把工作内存中的变量传输给执行引擎，每当虚拟机遇到一个需要使用到变量的值，就会使用到这个指令
  - assign （赋值）：作用于工作内存中的变量，它把一个从执行引擎中接受到的值放入工作内存的变量副本中
  - store  （存储）：作用于主内存中的变量，它把一个从工作内存中一个变量的值传送到主内存中，以便后续的write使用
  - write 　（写入）：作用于主内存中的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中

　　JMM对这八种指令的使用，制定了如下规则：

- - 不允许read和load、store和write操作之一单独出现。即使用了read必须load，使用了store必须write
  - 不允许线程丢弃他最近的assign操作，即工作变量的数据改变了之后，必须告知主存
  - 不允许一个线程将没有assign的数据从工作内存同步回主内存
  - 一个新的变量必须在主内存中诞生，不允许工作内存直接使用一个未被初始化的变量。就是怼变量实施use、store操作之前，必须经过assign和load操作
  - 一个变量同一时间只有一个线程能对其进行lock。多次lock后，必须执行相同次数的unlock才能解锁
  - 如果对一个变量进行lock操作，会清空所有工作内存中此变量的值，在执行引擎使用这个变量前，必须重新load或assign操作初始化变量的值
  - 如果一个变量没有被lock，就不能对其进行unlock操作。也不能unlock一个被其他线程锁住的变量
  - 对一个变量进行unlock操作之前，必须把此变量同步回主内存

　　JMM对这八种操作规则和对[volatile的一些特殊规则](https://www.cnblogs.com/null-qige/p/8569131.html)就能确定哪里操作是线程安全，哪些操作是线程不安全的了。但是这些规则实在复杂，很难在实践中直接分析。所以一般我们也不会通过上述规则进行分析。更多的时候，使用java的happen-before规则来进行分析。

## 　　模型特征

　　**原子性：**例如上面八项操作，在操作系统里面是不可分割的单元。被synchronized关键字或其他锁包裹起来的操作也可以认为是原子的。从一个线程观察另外一个线程的时候，看到的都是一个个原子性的操作。

```java
1         synchronized (this) {
2             a=1;
3             b=2;
4         }
```

　　例如一个线程观察另外一个线程执行上面的代码，只能看到a、b都被赋值成功结果，或者a、b都尚未被赋值的结果。

　　**可见性：**每个工作线程都有自己的工作内存，所以当某个线程修改完某个变量之后，在其他的线程中，未必能观察到该变量已经被修改。volatile关键字要求被修改之后的变量要求立即更新到主内存，每次使用前从主内存处进行读取。因此volatile可以保证可见性。除了volatile以外，synchronized和final也能实现可见性。synchronized保证unlock之前必须先把变量刷新回主内存。final修饰的字段在构造器中一旦完成初始化，并且构造器没有this逸出，那么其他线程就能看到final字段的值。

　　**有序性：**java的有序性跟线程相关。如果在线程内部观察，会发现当前线程的一切操作都是有序的。如果在线程的外部来观察的话，会发现线程的所有操作都是无序的。因为JMM的工作内存和主内存之间存在延迟，而且java会对一些指令进行重新排序。volatile和synchronized可以保证程序的有序性，很多程序员只理解这两个关键字的执行互斥，而没有很好的理解到volatile和synchronized也能保证指令不进行重排序。

## 　　Volatile内存语义

　　　[volatile的一些特殊规则](https://www.cnblogs.com/null-qige/p/8569131.html)

## 　　Final域的内存语义

　　被final修饰的变量，相比普通变量，内存语义有一些不同。具体如下：

- - JMM禁止把Final域的写重排序到构造器的外部。
  - 在一个线程中，初次读该对象和读该对象下的Final域，JMM禁止处理器重新排序这两个操作。

```java
 1 public class FinalConstructor {
 2 
 3     final int a;
 4 
 5     int b;
 6 
 7     static FinalConstructor finalConstructor;
 8 
 9     public FinalConstructor() {
10         a = 1;
11         b = 2;
12     }
13 
14     public static void write() {
15         finalConstructor = new FinalConstructor();
16     }
17 
18     public static void read() {
19         FinalConstructor constructor = finalConstructor;
20         int A = constructor.a;
21         int B = constructor.b;
22     }
23 }
```



　　假设现在有线程A执行FinalConstructor.write()方法，线程B执行FinalConstructor.read()方法。

　　对应上述的Final的第一条规则，因为JMM禁止把Final域的写重排序到构造器的外部，而对普通变量没有这种限制，所以变量A=1，而变量B可能会等于2（构造完成），也有可能等于0（第11行代码被重排序到构造器的外部）。

 　对应上述的Final的第二条规则，如果constructor的引用不为null，A必然为1，要么constructor为null，抛出空指针异常。保证读final域之前，一定会先读该对象的引用。但是普通对象就没有这种规则。

　　（上述的Final规则反复测试，遗憾的是我并没有能模拟出来普通变量不能正常构造的结果）

## 　　Happen-Before（先行发生规则）

　　在常规的开发中，如果我们通过上述规则来分析一个并发程序是否安全，估计脑壳会很疼。因为更多时候，我们是分析一个并发程序是否安全，其实都依赖Happen-Before原则进行分析。Happen-Before被翻译成先行发生原则，意思就是当A操作先行发生于B操作，则在发生B操作的时候，操作A产生的影响能被B观察到，“影响”包括修改了内存中的共享变量的值、发送了消息、调用了方法等。

　　Happen-Before的规则有以下几条

- - 程序次序规则（Program Order Rule）：在一个线程内，程序的执行规则跟程序的书写规则是一致的，从上往下执行。
  - 管程锁定规则（Monitor Lock Rule）：一个Unlock的操作肯定先于下一次Lock的操作。这里必须是同一个锁。同理我们可以认为在synchronized同步同一个锁的时候，锁内先行执行的代码，对后续同步该锁的线程来说是完全可见的。
  - volatile变量规则（volatile Variable Rule）：对同一个volatile的变量，先行发生的写操作，肯定早于后续发生的读操作
  - 线程启动规则（Thread Start Rule）：Thread对象的start()方法先行发生于此线程的没一个动作
  - 线程中止规则（Thread Termination Rule）：Thread对象的中止检测（如：Thread.join()，Thread.isAlive()等）操作，必行晚于线程中所有操作
  - 线程中断规则（Thread Interruption Rule）：对线程的interruption（）调用，先于被调用的线程检测中断事件(Thread.interrupted())的发生
  - 对象中止规则（Finalizer Rule）：一个对象的初始化方法先于一个方法执行Finalizer()方法
  - 传递性（Transitivity）：如果操作A先于操作B、操作B先于操作C,则操作A先于操作C

　　以上就是Happen-Before中的规则。通过这些条件的判定，仍然很难判断一个线程是否能安全执行，毕竟在我们的时候线程安全多数依赖于工具类的安全性来保证。想提高自己对线程是否安全的判断能力，必然需要理解所使用的框架或者工具的实现，并积累线程安全的经验。