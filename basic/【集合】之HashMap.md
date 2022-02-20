# 数据结构

## Entry类型

​	HashMap主要维护的是一个Entry类型的的数组来存储数据

​	Entry包括key/value/hash/next，其中key是final修饰的

​	1.8之后使用Node替代Entry，实际效果是一样的

## 哈希表

​	哈希表（数组+链表），是根据key直接访问在内存存储位置的数据结构，也就是通过hash函数将所需查询的数据映射到表中的一个位置来访问记录，映射函数叫做散列函数，存放记录的数组叫做散列表。ps:通过将key通过固定的算法函数（hash函数）转换成一个整型数字，然后对该数字对数组长度进行取余，取余结果就当作数组的下标，将value存储在该数字为下标的数组空间里

## 参数变量

### 	约定

​		约定前面的数组结构的每一个格格成为桶bucket

​		约定桶后面存放的每一个数据成为bin（bin这个术语来自jdk1.8）

### SIZE

​		size表示HashMap中存放的KV的数量（为链表和数中的KV的总和）size == threshold * loadFactor

### CAPACITY

​		capacity译为容量，capacity就是指HashMap中桶的数量，默认为16，一般第一次扩容为64，之后是2倍，总之都是2的幂方，当初始化时设置容量大小，构造方法 HashMap(int initialCapacity，float loadFactor)，HashMap 并不是直接使用外部传递进来的 initialCapacity，而是经过了 tableSizeFor() 方法的处理，再赋值到 threshole 上，在 tableSizeFor() 方法中，通过逐步位运算，就可以让返回值，保持在 2 的 N 次幂。以方便在扩容的时候，快速计算数据在扩容后的新表中的位置

### LOADFACTOR

​		 loadFactor译为加载因子，用来衡量HashMap满的程度，默认指0.75f，计算HashMap的实时装载因子的方法为：size/capacity，而不是占用桶的数量去除以capacity，负载因子越大，对空间的利用更充分，然后后果是查找效率的降低；如果负载因子太小，那么散列表的数据将过于稀疏，对空间造成严重浪费。系统默认负载因子为0.75。

### THRESHOLD

​		表示当HashMap的Size大于threshold会执行resize操作，threshold=capacity*loadFactor

# HASH冲突

## 		什么是hash冲突

​		对不同的关键字可能得到同一散列地址，即k1≠k2，而f(k1)=f(k2)，或f(k1) MOD 容量 =f(k2) MOD 容量，这种现象称为碰撞，亦称冲突。
通过构造性能良好的hash函数，可以减少冲突，但一般不可能完全避免冲突，因此解决冲突是hash表的另一个关键问题。
创建和查找hash表都会遇到冲突，两种情况下解决冲突的方法应该一致

## 解决hash冲突

​		HashMap里面的bucket出现了单链表的形式，散列表要解决的一个问题就是散列值的冲突问题，通常是两种方法：链表法和开放地址法。链表法就是将相同hash值的对象组织成一个链表放在hash值对应的槽位；开放地址法是通过一个探测算法，当某个槽位已经被占据的情况下继续查找下一个可以使用的槽位。java.util.HashMap采用的链表法的方式，链表是单向链表，形成单链表的核心方法-->addEntry()，首先判断是否需要扩容，需要则扩充HashMap的数据结果为原来的两倍，然后将新添加的 Entry 对象放入 table 数组的 bucketIndex 索引处——如果 bucketIndex 索引处已经有了一个 Entry 对象，那新添加的 Entry 对象指向原有的 Entry 对象(产生一个 Entry 链)，如果 bucketIndex 索引处没有 Entry 对象，也就是上面程序代码的 e 变量是 null，也就是新放入的 Entry 对象指向 null，也就是没有产生 Entry 链。 HashMap里面没有出现hash冲突时，没有形成单链表时，hashmap查找元素很快,get()方法能够直接定位到元素，但是出现单链表后，单个bucket 里存储的不是一个 Entry，而是一个 Entry 链，系统只能必须按顺序遍历每个 Entry，直到找到想搜索的 Entry 为止——如果恰好要搜索的 Entry 位于该 Entry 链的最末端(该 Entry 是最早放入该 bucket 中)，那系统必须循环到最后才能找到该元素

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
        	// 如果其大小超过threshold 并且table[index]的位置不为空
        	// 扩充HashMap的数据结果为原来的两倍
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
}
```



# 扩容

​		核心代码

```java
void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
        	//capacity 容量
            threshold = Integer.MAX_VALUE;
            return;
        }
		// 创建一个新的对象
        Entry[] newTable = new Entry[newCapacity];
        // 将旧的对象内的值赋值到新的对象中
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        // 重新赋值新的临界点
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
    
    void transfer(Entry[] newTable, boolean rehash) {
    	// 新table的长度
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

主要的resize()方法的转换操作都在transfer()方法内.我们详细的画图了解下这个循环.

1、for (Entry<K,V> e : table) {}外层循环,循环遍历旧table的每一个值.这不难理解,因为要将旧table内的值拷贝进新table.
2、链表拷贝操作.主要是四行代码.e.next = newTable[i]; newTable[i] = e; e = next;

1 、next = e.next,目的是记录下原始的e.next的地址.图上为Entry-1-1
2 、e.next = newTable[i],也就是将newTable[3]替代了原来的entry1-1.目的是为了记录原来的newTable[3]的链表头.
3 、newTable[i] = e,也就是将entry1-0替换成新的链表头.

4、e = next;,循环遍历指针.将e = entry1-1.开始处理entry1-1.

由上述的插入过程,我们可以看出.这是一个倒叙的链表插入过程.
(比如 1-0 -> 1-1 ->1-2 插入后将变为 1-2 -> 1-1 -> 1-0)

# 线程安全

​		在扩容时，存储在LinkedList中的元素的次序会反过来，因为移动到新的bucket位置的时候，HashMap并不会将元素放在LinkedList的尾部，而是放在头部，这是为了避免尾部遍历(tail traversing)，因为倒叙链表插入的情况,导致HashMap在resize()的情况下,导致链表出现环的出现.一旦出现了环那么在while(null != p.next){}的循环的时候.就会出现死循环导致线程阻塞.那么,多线程的时候,为什么会出现环状呢？当多个线程同时操作的时候，就会出现竞态条件同一行号的代码会同时被两个执行，从而出现环状

# PUT/GET流程

## PUT

```java
public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
        	// 如果table为空的table,我们需要对其进行初始化操作.
            inflateTable(threshold);
        }
        if (key == null)
        	// 如果key为null的话 我们对其进行特殊操作(其实是放在table[0])
            return putForNullKey(value);
        // 计算hash值
        int hash = hash(key);
        // 根据hash值获取其在table数组内的位置.
        int i = indexFor(hash, table.length);
        // 遍历循环链表(结构图类似上图)
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 如果找到其值的话,将原来的值进行覆盖(满足Map数据类型的特性.)
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
		// 如果都没有找到相同都key的值, 那么这个值是一个新值.(直接进行插入操作.)
        modCount++;
    	//解决hash冲突，判断是否需要扩容，并判断是否树化
        addEntry(hash, key, value, i);
        return null;
    }



```

<img src="/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20210622084313967.png" alt="image-20210622084313967" style="zoom:80%;" />

​	inflateTable这个方法用于为主干数组table在内存中分配存储空间，通过roundUpToPowerOf2(toSize)可以确保capacity为大于或等于toSize的最接近toSize的二次幂，比如toSize=13,则capacity=16;to_size=16,capacity=16;to_size=17,capacity=32.


```java
private static int roundUpToPowerOf2(int number) {
        // assert number >= 0 : "number must be non-negative";
        return number >= MAXIMUM_CAPACITY
                ? MAXIMUM_CAPACITY
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }
```

​	roundUpToPowerOf2中的这段处理使得数组长度一定为2的次幂，Integer.highestOneBit是用来获取最左边的bit（其他bit位为0）所代表的数值.
​	hash函数
//这是一个神奇的函数，用了很多的异或，移位等运算，对key的hashcode进一步进行计算以及二进制位的调整等来保证最终获取的存储位置尽量分布均匀

```java
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }  
  	h ^= k.hashCode();
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
以上hash函数计算出的值，通过indexFor进一步处理来获取实际的存储位置
　

```java
　/**
     * 返回数组下标
     */
    static int indexFor(int h, int length) {
        return h & (length-1);
    }
```

h&（length-1）保证获取的index一定在数组范围内，举个例子，默认容量16，length-1=15，h=18,转换成二进制计算为

```java
     1  0  0  1  0
 &   0  1  1  1  1
__________________
     0  0  0  1  0    = 2
```

　　最终计算出的index=2。有些版本的对于此处的计算会使用 取模运算，也能保证index一定在数组范围内，不过位运算对计算机来说，性能更高一些（HashMap中有大量位运算）
所以最终存储位置的确定流程是这样的：

![image-20210720104200503](/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20210720104200503.png)

再来看看addEntry的实现：


```java
void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);//当size超过临界阈值threshold，并且即将发生哈希冲突时进行扩容
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
    createEntry(hash, key, value, bucketIndex);
}
```
　　通过以上代码能够得知，当发生哈希冲突并且size大于阈值的时候，需要进行数组扩容，扩容时，需要新建一个长度为之前数组2倍的新的数组，然后将当前的Entry数组中的元素全部传输过去，扩容后的新数组长度为之前的2倍，所以扩容相对来说是个耗资源的操作。
三、为何HashMap的数组长度一定是2的次幂？
我们来继续看上面提到的resize方法

```java
     void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
如果数组进行扩容，数组长度发生变化，而存储位置 index = h&(length-1),index也可能会发生变化，需要重新计算index，我们先来看看transfer这个方法

```java
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
　　　　　//for循环中的代码，逐个遍历链表，重新计算索引位置，将老数组数据复制到新数组中去（数组不存储实际数据，所以仅仅是拷贝引用而已）
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
　　　　　　　　　 //将当前entry的next链指向新的索引位置,newTable[i]有可能为空，有可能也是个entry链，如果是entry链，直接在链表头部插入。
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

　　这个方法将老数组中的数据逐个链表地遍历，扔到新的扩容后的数组中，我们的数组索引位置的计算是通过 对key值的hashcode进行hash扰乱运算后，再通过和 length-1进行位运算得到最终数组索引位置。
　　hashMap的数组长度一定保持2的次幂，比如16的二进制表示为 10000，那么length-1就是15，二进制为01111，同理扩容后的数组长度为32，二进制表示为100000，length-1为31，二进制表示为011111。从下图可以我们也能看到这样会保证低位全为1，而扩容后只有一位差异，也就是多出了最左位的1，这样在通过 h&(length-1)的时候，只要h对应的最左边的那一个差异位为0，就能保证得到的新的数组索引和老数组索引一致(大大减少了之前已经散列良好的老数组的数据位置重新调换)，个人理解。
![image-20210720104229387](/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20210720104229387.png)

 还有，数组长度保持2的次幂，length-1的低位都为1，会使得获得的数组索引index更加均匀，比如：

![image-20210720104250304](/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20210720104250304.png)

　　我们看到，上面的&运算，高位是不会对结果产生影响的（hash函数采用各种位运算可能也是为了使得低位更加散列），我们只关注低位bit，如果低位全部为1，那么对于h低位部分来说，任何一位的变化都会对结果产生影响，也就是说，要得到index=21这个存储位置，h的低位只有这一种组合。这也是数组长度设计为必须为2的次幂的原因。

![image-20210720104314015](/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20210720104314015.png)

　　如果不是2的次幂，也就是低位不是全为1此时，要使得index=21，h的低位部分不再具有唯一性了，哈希冲突的几率会变的更大，同时，index对应的这个bit位无论如何不会等于1了，而对应的那些数组位置也就被白白浪费了



java7：新值插入到链表的最前面，先（判断）扩容后插入新值。
java8：新值插入到链表的最后面，先插值再（判断）扩容

GET

 1. 对key进行null检查。如果key是null，table[0]这个位置的元素将被返回。

 2. key的hashcode()方法被调用，然后计算hash值。

 3. indexFor(hash,table.length)用来计算要获取的Entry对象在table数组中的精确的位置，使用刚才计算的hash值。

 4. 在获取了table数组的索引之后，会迭代链表，调用equals()方法检查key的相等性，如果equals()方法返回true，get方法返回Entry对象的value，否则，产生Entry链返回false，不相等的话判断该节点是否为树节点，是的话以树节点方式遍历整棵树来查找，不是的话那就说明存储结构是链表，已遍历链表的方式查找

# 树化条件

## 条件

​		链表长度大于8

​		数组长度大于64

## 为什么要树化

​		主要是为了避免哈希碰撞拒绝服务攻击

​		当向服务器一次提交数万个hashcode相同的字符串，服务器的查询时间过长，让服务器的CPU被大量占用，当有其他更多的请求时服务器会拒绝服务。而使用红黑树可以将查询时间降低到一定的数量级，可以有效避免哈希碰撞拒绝服务攻击

