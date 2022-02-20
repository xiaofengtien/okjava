垃圾收集 Garbage Collection 通常被称为“GC”，本文详细讲述Java垃圾回收机制。

> 导读：
>
> 1、什么是GC
>
> 2、GC常用算法
>
> 3、垃圾收集器
>
> 4、finalize()方法详解
>
> 5、总结--根据GC原理来优化代码

正式阅读之前需要了解相关概念：

Java 堆内存分为新生代和老年代，新生代中又分为1个 Eden 区域 和 2个 Survivor 区域。

## 一、什么是GC：

每个程序员都遇到过内存溢出的情况，程序运行时，内存空间是有限的，那么如何及时的把不再使用的对象清除将内存释放出来，这就是GC要做的事。

理解GC机制就从：“GC的区域在哪里”，“GC的对象是什么”，“GC的时机是什么”，“GC做了哪些事”几方面来分析。 

### 1、需要GC的内存区域

jvm 中，程序计数器、虚拟机栈、本地方法栈都是随线程而生随线程而灭，栈帧随着方法的进入和退出做入栈和出栈操作，实现了自动的内存清理，因此，我们的内存垃圾回收主要集中于 java 堆和方法区中，在程序运行期间，这部分内存的分配和使用都是动态的。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE4NTI1NTQ5My00NzgxMjM0MDkucG5n?x-oss-process=image/format,png)

 

### 2、GC的对象

需要进行回收的对象就是已经没有存活的对象，判断一个对象是否存活常用的有两种办法：引用计数和可达分析。

（1）引用计数：每个对象有一个引用计数属性，新增一个引用时计数加1，引用释放时计数减1，计数为0时可以回收。此方法简单，无法解决对象相互循环引用的问题。

（2）可达性分析（Reachability Analysis）：从GC Roots开始向下搜索，搜索所走过的路径称为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的。不可达对象。

> 在Java语言中，GC Roots包括：
>
> 虚拟机栈中引用的对象。
>
> 方法区中类静态属性实体引用的对象。
>
> 方法区中常量引用的对象。
>
> 本地方法栈中JNI引用的对象。

### 3、什么时候触发GC

(1)程序调用System.gc时可以触发

(2)系统自身来决定GC触发的时机（根据Eden区和From Space区的内存大小来决定。当内存大小不足时，则会启动GC线程并停止应用线程）

> GC又分为 minor GC 和 Full GC (也称为 Major GC )
>
> Minor GC触发条件：当Eden区满时，触发Minor GC。
>
> Full GC触发条件：
>
>   a.调用System.gc时，系统建议执行Full GC，但是不必然执行
>
>   b.老年代空间不足
>
>   c.方法去空间不足
>
>   d.通过Minor GC后进入老年代的平均大小大于老年代的可用内存
>
>   e.由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

 

### 4、GC做了什么事

 主要做了清理对象，整理内存的工作。Java堆分为新生代和老年代，采用了不同的回收方式。（回收方式即回收算法详见后文）

### 5、判断对象不被使用的算法

#### 引用计数法

引用计数法(Reference Counting):给对象添加一个引用计数器，每当有一个地方引用它时，计数器值就加1， 当引用失效时，计数器值就减1，任何时刻计数器为0的对象就是不可能被再使用的。

> 但这种方法算法很难解决对象之间相互循环引用的问题。举个例子
>
> 对象objA和objB都有字段instance，赋值令objA.instance=objB，以及objB.instance=objA，除此之外这2个 对象再无任何引用，实际上这2个对象已经不可能再被访问，但是他们因为互相引用这对方，导致他们的计数 器都不为0，于是引用计数法无法通知GC收集器回收他们

```java
public ReferenceCountingGC(String name){
}
public static void testGC(){
    ReferenceCountingGC a = new ReferenceCountingGC("objA");
    ReferenceCountingGC b = new ReferenceCountingGC("objB");
    a.instance = b;
    b.instance = a;
a = null;
b = null; }
```

#### 可达性分析法

可达性分析算法(Reachability Analysis):通过一些被称为引用链(GC Roots)的对象作为起点，从这些节点开 始向下搜索，搜索走过的路径被称为(Reference Chain)，当一个对象到 GC Roots 没有任何引用链相连时(即从 GC Roots 节点到该节点不可达)，则证明该对象是不可用的。

<img src="/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20211117145508735.png" alt="image-20211117145508735" style="zoom:25%;" />

如图，object5，object6和object7虽然互相有关联，但是他们到GC Roots是不可达的，所以他们会被判定为是可 回收的对象。

所以什么样的对象需要回收呢?

对象到GC Root没有引用链，那么这个对象不可用，需要回收。Java就是用可达性分析法来判断对象是否需要被回收的

#### 可作为**GC Roots**的对象

> 1. 虚拟机栈(栈帧中的本地变量表)中引用的对象 
> 2. 方法区中类静态属性引用的对象
> 3. 方法区中常量引用的对象
> 4. 本地方法栈中JNI(Native方法)引用的对象

##### 虚拟机栈(栈帧中的本地变量表)中引用的对象

此时的 s，即为 GC Root，当s置空时，localParameter 对象也断掉了与 GC Root 的引用链，将被回收。

```java
public class StackLocalParameter {
    public StackLocalParameter(String name){}
}
public static void testGC(){
    StackLocalParameter s = new StackLocalParameter("localParameter");
    s = null;
}
```

##### 方法区中类静态属性引用的对象

s 为 GC Root，s 置为 null，经过 GC 后，s 所指向的 properties 对象由于无法与 GC Root 建立关系被回收。
 而 m 作为类的静态属性，也属于 GC Root，parameter 对象依然与 GC root 建立着连接，所以此时 parameter 对象并不会被回收。

```java
public class MethodAreaStaicProperties {
    public static MethodAreaStaicProperties m;
    public MethodAreaStaicProperties(String name){}
} 
public static void testGC(){
    MethodAreaStaicProperties s = new MethodAreaStaicProperties("properties");
    s.m = new MethodAreaStaicProperties("parameter");
    s = null;
}
```

##### 方法区中常量引用的对象

m 即为方法区中的常量引用，也为 GC Root，s 置为 null 后，final 对象也不会因没有与 GC Root 建立联系而被回 收。

```java
public class MethodAreaStaicProperties {
    public static final MethodAreaStaicProperties m =
MethodAreaStaicProperties("final");
    public MethodAreaStaicProperties(String name){}
}
public static void testGC(){
    MethodAreaStaicProperties s = new MethodAreaStaicProperties("staticProperties");
    s = null;
}
```

##### 本地方法栈中引用的对象

> 和虚拟机栈类似，一个是虚拟机层面的调用，一个是本地层面的调用

##  二、GC常用算法

GC常用算法有：标记-清除算法，标记-压缩算法，复制算法，分代收集算法。

目前主流的JVM（HotSpot）采用的是分代收集算法。

###  1、标记-清除算法

为每个对象存储一个标记位，记录对象的状态（活着或是死亡）。分为两个阶段，一个是标记阶段，这个阶段内，为每个对象更新标记位，检查对象是否死亡；第二个阶段是清除阶段，该阶段对死亡的对象进行清除，执行 GC 操作。

> 优点
> 最大的优点是，标记—清除算法中每个活着的对象的引用只需要找到一个即可，找到一个就可以判断它为活的。此外，更重要的是，这个算法并不移动对象的位置。
>
> 缺点
> 它的缺点就是效率比较低（递归与全堆对象遍历）。每个活着的对象都要在标记阶段遍历一遍；所有对象都要在清除阶段扫描一遍，因此算法复杂度较高。没有移动对象，导致可能出现很多碎片空间无法利用的情况。

图例

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5MjM0NDc3Ni0xNDU2OTIxNTU2LnBuZw?x-oss-process=image/format,png)

###  2、标记-压缩算法（标记-整理）

标记-压缩法是标记-清除法的一个改进版。同样，在标记阶段，该算法也将所有对象标记为存活和死亡两种状态；不同的是，在第二个阶段，该算法并没有直接对死亡的对象进行清理，而是将所有存活的对象整理一下，放到另一处空间，然后把剩下的所有对象全部清除。这样就达到了标记-整理的目的。

> 优点
> 该算法不会像标记-清除算法那样产生大量的碎片空间。
> 缺点
> 如果存活的对象过多，整理阶段将会执行较多复制操作，导致算法效率降低。
> 图例.      

​          ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5MjgwODIwOS0xMTQ0MDExNjcxLnBuZw?x-oss-process=image/format,png)     

左边是标记阶段，右边是整理之后的状态。可以看到，该算法不会产生大量碎片内存空间。

###  3、复制算法

该算法将内存平均分成两部分，然后每次只使用其中的一部分，当这部分内存满的时候，将内存中所有存活的对象复制到另一个内存中，然后将之前的内存清空，只使用这部分内存，循环下去。

注意： 
这个算法与标记-整理算法的区别在于，该算法不是在同一个区域复制，而是将所有存活的对象复制到另一个区域内。

> 优点
>
> 实现简单；不产生内存碎片
>
> 缺点
> 每次运行，总有一半内存是空的，导致可使用的内存空间只有原来的一半。

图例

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5MzEyMDc5Ny02OTQ3ODYxMTUucG5n?x-oss-process=image/format,png)

### 4、分代收集算法

现在的虚拟机垃圾收集大多采用这种方式，它根据对象的生存周期，将堆分为新生代(Young)和老年代(Tenure)。在新生代中，由于对象生存期短，每次回收都会有大量对象死去，那么这时就采用复制算法。老年代里的对象存活率较高，没有额外的空间进行分配担保，所以可以使用标记-整理 或者 标记-清除。

 具体过程：新生代(Young)分为Eden区，From区与To区

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NDAzMzA1MC0xNTU0OTMzMDkzLnBuZw?x-oss-process=image/format,png)

当系统创建一个对象的时候，总是在Eden区操作，当这个区满了，那么就会触发一次YoungGC，也就是年轻代的垃圾回收。一般来说这时候不是所有的对象都没用了，所以就会把还能用的对象复制到From区。 

​          ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NDEzNjA5Ny05MzYzNTg2NTcucG5n?x-oss-process=image/format,png)  

这样整个Eden区就被清理干净了，可以继续创建新的对象，当Eden区再次被用完，就再触发一次YoungGC，然后呢，注意，这个时候跟刚才稍稍有点区别。这次触发YoungGC后，会将Eden区与From区还在被使用的对象复制到To区， 

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NDIwNzYwNi0yOTg1Nzc2ODIucG5n?x-oss-process=image/format,png)

再下一次YoungGC的时候，则是将Eden区与To区中的还在被使用的对象复制到From区。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NDIzNjAwNy0xNDI4NDA4MzUyLnBuZw?x-oss-process=image/format,png)

经过若干次YoungGC后，有些对象在From与To之间来回游荡，这时候From区与To区亮出了底线（阈值），这些家伙要是到现在还没挂掉，对不起，一起滚到（复制）老年代吧。 

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NDMwMjk3NC0xODQ0NzM0MzMxLnBuZw?x-oss-process=image/format,png)

老年代经过这么几次折腾，也就扛不住了（空间被用完），好，那就来次集体大扫除（Full GC），也就是全量回收。如果Full GC使用太频繁的话，无疑会对系统性能产生很大的影响。所以要合理设置年轻代与老年代的大小，尽量减少Full GC的操作。

## 三、垃圾收集器

如果说收集算法是内存回收的方法论，垃圾收集器就是内存回收的具体实现

### Serial收集器

串行收集器是最古老，最稳定以及效率高的收集器
可能会产生较长的停顿，只使用一个线程去回收

```properties
-XX:+UseSerialGC
```

> 新生代、老年代使用串行回收
> 新生代复制算法
> 老年代标记-压缩

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NTMwNDU0Ny0xOTc5MTU2OTc3LnBuZw?x-oss-process=image/format,png)

### 并行收集器

###  ParNew

```properties
-XX:+UseParNewGC（new代表新生代，所以适用于新生代）
```



> 新生代并行
> 老年代串行

​		Serial收集器新生代的并行版本
​		在新生代回收时使用复制算法
​		多线程，需要多核支持
​		-XX:ParallelGCThreads 限制线程数量

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NTMzNzMzNi0xNzc3MDAwNzk4LnBuZw?x-oss-process=image/format,png)

 

###  Parallel收集器

​	  类似ParNew 
​	  新生代复制算法 
​     老年代标记-压缩 
​     更加关注吞吐量 

```properties
-XX:+UseParallelGC  
```

使用Parallel收集器+ 老年代串行

```properties
-XX:+UseParallelOldGC 
```

使用Parallel收集器+ 老年代并行

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NTQwNDI1MC0yMTQyOTA5NTUzLnBuZw?x-oss-process=image/format,png)


2.3 其他GC参数

-XX:MaxGCPauseMills

最大停顿时间，单位毫秒
GC尽力保证回收时间不超过设定值
-XX:GCTimeRatio 

0-100的取值范围
垃圾收集时间占总时间的比
默认99，即最大允许1%时间做GC
这两个参数是矛盾的。因为停顿时间和吞吐量不可能同时调优

### CMS收集器

> Concurrent Mark Sweep 并发标记清除（应用程序线程和GC线程交替执行）
> 使用标记-清除算法
> 并发阶段会降低吞吐量（停顿时间减少，吞吐量降低）
> 老年代收集器（新生代使用ParNew）
> -XX:+UseConcMarkSweepGC

CMS运行过程比较复杂，着重实现了标记的过程，可分为

1.初始标记（会产生全局停顿）

​		根可以直接关联到的对象
​		速度快

​		2.并发标记（和用户线程一起） 

​				主要标记过程，标记全部对象

​		3.重新标记 （会产生全局停顿） 

​				由于并发标记时，用户线程依然运行，因此在正式清理前，再做修正

​		4.并发清除（和用户线程一起） 

​				基于标记结果，直接清理对象

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NTQ0NzY5OC01OTcyNzc1OTUucG5n?x-oss-process=image/format,png)


这里就能很明显的看出，为什么CMS要使用标记清除而不是标记压缩，如果使用标记压缩，需要多对象的内存位置进行改变，这样程序就很难继续执行。但是标记清除会产生大量内存碎片，不利于内存分配。 

CMS收集器特点：

> 尽可能降低停顿
>
> 会影响系统整体吞吐量和性能

比如，在用户线程运行过程中，分一半CPU去做GC，系统性能在GC阶段，反应速度就下降一半
清理不彻底 

因为在清理阶段，用户线程还在运行，会产生新的垃圾，无法清理
因为和用户线程一起运行，不能在空间快满时再清理（因为也许在并发GC的期间，用户线程又申请了大量内存，导致内存不够） 

```properties
-XX:CMSInitiatingOccupancyFraction设置触发GC的阈值
```

如果不幸内存预留空间不够，就会引起concurrent mode failure
一旦 concurrent mode failure产生，将使用串行收集器作为后备。

CMS也提供了整理碎片的参数：

```properties
-XX:+ UseCMSCompactAtFullCollection Full GC后，进行一次整理
```

整理过程是独占的，会引起停顿时间变长

```properties
-XX:+CMSFullGCsBeforeCompaction  
```

设置进行几次Full GC后，进行一次碎片整理

```
-XX:ParallelCMSThreads 
```

设定CMS的线程数量（一般情况约等于可用CPU数量）
CMS的提出是想改善GC的停顿时间，在GC过程中的确做到了减少GC时间，但是同样导致产生大量内存碎片，又需要消耗大量时间去整理碎片，从本质上并没有改善时间。  

### G1收集器

G1是目前技术发展的最前沿成果之一，HotSpot开发团队赋予它的使命是未来可以替换掉JDK1.5中发布的CMS收集器。

#### 与CMS收集器相比G1收集器有以下特点：

(1) 空间整合，G1收集器采用标记整理算法，不会产生内存空间碎片。分配大对象时不会因为无法找到连续空间而提前触发下一次GC。

(2)可预测停顿，这是G1的另一大优势，降低停顿时间是G1和CMS的共同关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒，这几乎已经是实时Java（RTSJ）的垃圾收集器的特征了。

上面提到的垃圾收集器，收集的范围都是整个新生代或者老年代，而G1不再是这样。使用G1收集器时，Java堆的内存布局与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔阂了，它们都是一部分（可以不连续）Region的集合。

G1的新生代收集跟ParNew类似，当新生代占用达到一定比例的时候，开始出发收集。

和CMS类似，G1收集器收集老年代对象会有短暂停顿。

步骤：

> (1)标记阶段，首先初始标记(Initial-Mark),这个阶段是停顿的(Stop the World Event)，并且会触发一次普通Mintor GC。对应GC log:GC pause (young) (inital-mark)
>
> (2)Root Region Scanning，程序运行过程中会回收survivor区(存活到老年代)，这一过程必须在young GC之前完成。
>
> (3)Concurrent Marking，在整个堆中进行并发标记(和应用程序并发执行)，此过程可能被young GC中断。在并发标记阶段，若发现区域对象中的所有对象都是垃圾，那个这个区域会被立即回收(图中打X)。同时，并发标记过程中，会计算每个区域的对象活性(区域中存活对象的比例)。![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NTc0ODg4NS02MzUzOTQ0MjkucG5n?x-oss-process=image/format,png) 
>
> (4)Remark, 再标记，会有短暂停顿(STW)。再标记阶段是用来收集 并发标记阶段 产生新的垃圾(并发阶段和应用程序一同运行)；G1中采用了比CMS更快的初始快照算法:snapshot-at-the-beginning (SATB)。
>
> (5)Copy/Clean up，多线程清除失活对象，会有STW。G1将回收区域的存活对象拷贝到新区域，清除Remember Sets，并发清空回收区域并把它返回到空闲区域链表中。
>
> ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjE5NTgxODgyMi04MTEyMDAxNTkucG5n?x-oss-process=image/format,png)
>
>  
>
> (6)复制/清除过程后。回收区域的活性对象已经被集中回收到深蓝色和深绿色区域。
>
>  

## 四、finalize()方法详解

1. ### finalize的作用

> (1)finalize()是Object的protected方法，子类可以覆盖该方法以实现资源清理工作，GC在回收对象之前调用该方法。
> (2)finalize()与C++中的析构函数不是对应的。C++中的析构函数调用的时机是确定的（对象离开作用域或delete掉），但Java中的finalize的调用具有不确定性
> (3)不建议用finalize方法完成“非内存资源”的清理工作，但建议用于：① 清理本地对象(通过JNI创建的对象)；② 作为确保某些非内存资源(如Socket、文件等)释放的一个补充：在finalize方法中显式调用其他资源释放方法。其原因可见下文[finalize的问题]


2. ### finalize的问题
   
   > (1)一些与finalize相关的方法，由于一些致命的缺陷，已经被废弃了，如System.runFinalizersOnExit()方法、Runtime.runFinalizersOnExit()方法
   > (2)System.gc()与System.runFinalization()方法增加了finalize方法执行的机会，但不可盲目依赖它们
   > (3)Java语言规范并不保证finalize方法会被及时地执行、而且根本不会保证它们会被执行
   > (4)finalize方法可能会带来性能问题。因为JVM通常在单独的低优先级线程中完成finalize的执行
   > (5)对象再生问题：finalize方法中，可将待回收对象赋值给GC Roots可达的对象引用，从而达到对象再生的目的
   > (6)finalize方法至多由GC执行一次(用户当然可以手动调用对象的finalize方法，但并不影响GC对finalize的行为)
   
3. ### finalize的执行过程(生命周期)

> (1) 首先，大致描述一下finalize流程：当对象变成(GC Roots)不可达时，GC会判断该对象是否覆盖了finalize方法，若未覆盖，则直接将其回收。否则，若对象未执行过finalize方法，将其放入F-Queue队列，由一低优先级线程执行该队列中对象的finalize方法。执行finalize方法完毕后，GC会再次判断该对象是否可达，若不可达，则进行回收，否则，对象“复活”。
> (2) 具体的finalize流程：
>   对象可由两种状态，涉及到两类状态空间，一是终结状态空间 F = {unfinalized, finalizable, finalized}；二是可达状态空间 R = {reachable, finalizer-reachable, unreachable}。各状态含义如下：
>
> unfinalized: 新建对象会先进入此状态，GC并未准备执行其finalize方法，因为该对象是可达的
> finalizable: 表示GC可对该对象执行finalize方法，GC已检测到该对象不可达。正如前面所述，GC通过F-Queue队列和一专用线程完成finalize的执行
> finalized: 表示GC已经对该对象执行过finalize方法
> reachable: 表示GC Roots引用可达
> finalizer-reachable(f-reachable)：表示不是reachable，但可通过某个finalizable对象可达
> unreachable：对象不可通过上面两种途径可达
> 状态变迁图：
>
> ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTQ4OTY2OS8yMDE4MTAvMTQ4OTY2OS0yMDE4MTAxNjIwMTAxMjIyMC03ODQwNDUxNDQucG5n?x-oss-process=image/format,png)



### 变迁说明：

>   (1)新建对象首先处于[reachable, unfinalized]状态(A)
>   (2)随着程序的运行，一些引用关系会消失，导致状态变迁，从reachable状态变迁到f-reachable(B, C, D)或unreachable(E, F)状态
>   (3)若JVM检测到处于unfinalized状态的对象变成f-reachable或unreachable，JVM会将其标记为finalizable状态(G,H)。若对象原处于[unreachable, unfinalized]状态，则同时将其标记为f-reachable(H)。
>   (4)在某个时刻，JVM取出某个finalizable对象，将其标记为finalized并在某个线程中执行其finalize方法。由于是在活动线程中引用了该对象，该对象将变迁到(reachable, finalized)状态(K或J)。该动作将影响某些其他对象从f-reachable状态重新回到reachable状态(L, M, N)
>   (5)处于finalizable状态的对象不能同时是unreahable的，由第4点可知，将对象finalizable对象标记为finalized时会由某个线程执行该对象的finalize方法，致使其变成reachable。这也是图中只有八个状态点的原因
>   (6)程序员手动调用finalize方法并不会影响到上述内部标记的变化，因此JVM只会至多调用finalize一次，即使该对象“复活”也是如此。程序员手动调用多少次不影响JVM的行为
>   (7)若JVM检测到finalized状态的对象变成unreachable，回收其内存(I)
>   (8)若对象并未覆盖finalize方法，JVM会进行优化，直接回收对象（O）
>   (9)注：System.runFinalizersOnExit()等方法可以使对象即使处于reachable状态，JVM仍对其执行finalize方法

 

## 五、总结 

> 根据GC的工作原理，我们可以通过一些技巧和方式，让GC运行更加有效率，更加符合应用程序的要求。一些关于程序设计的几点建议： 
>
> 1.最基本的建议就是尽早释放无用对象的引用。大多数程序员在使用临时变量的时候，都是让引用变量在退出活动域（scope）后，自动设置为 null.我们在使用这种方式时候，必须特别注意一些复杂的对象图，例如数组，队列，树，图等，这些对象之间有相互引用关系较为复杂。对于这类对象，GC 回收它们一般效率较低。如果程序允许，尽早将不用的引用对象赋为null.这样可以加速GC的工作。 
>
> 2.尽量少用finalize函数。finalize函数是Java提供给程序员一个释放对象或资源的机会。但是，它会加大GC的工作量，因此尽量少采用finalize方式回收资源。 
>
> 3.如果需要使用经常使用的图片，可以使用soft应用类型。它可以尽可能将图片保存在内存中，供程序调用，而不引起OutOfMemory. 
>
> 4.注意集合数据类型，包括数组，树，图，链表等数据结构，这些数据结构对GC来说，回收更为复杂。另外，注意一些全局的变量，以及一些静态变量。这些变量往往容易引起悬挂对象（dangling reference），造成内存浪费。 
>
> 5.当程序有一定的等待时间，程序员可以手动执行System.gc（），通知GC运行，但是Java语言规范并不保证GC一定会执行。使用增量式GC可以缩短Java程序的暂停时间。