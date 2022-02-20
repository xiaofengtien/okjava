### 过期的数据的删除策略了解么？

如果假设你设置了一批 key 只能存活 1 分钟，那么 1 分钟后，Redis 是怎么对这批 key 进行删除的呢？

常用的过期数据的删除策略就两个（重要！自己造缓存轮子的时候需要格外考虑的东西）：

1. **惰性删除** ：只会在取出 key 的时候才对数据进行过期检查。这样对 CPU 最友好，但是可能会造成太多过期 key 没有被删除。
2. **定期删除** ： 每隔一段时间抽取一批 key 执行删除过期 key 操作。并且，Redis 底层会通过限制删除操作执行的时长和频率来减少删除操作对 CPU 时间的影响。

定期删除对内存更加友好，惰性删除对 CPU 更加友好。两者各有千秋，所以 Redis 采用的是 **定期删除+惰性/懒汉式删除** 。

但是，仅仅通过给 key 设置过期时间还是有问题的。因为还是可能存在定期删除和惰性删除漏掉了很多过期 key 的情况。这样就导致大量过期 key 堆积在内存里，然后就 Out of memory 了。

怎么解决这个问题呢？答案就是： **Redis 内存淘汰机制。**

### Redis 内存淘汰机制了解么？

> 相关问题：MySQL 里有 2000w 数据，Redis 中只存 20w 的数据，如何保证 Redis 中的数据都是热点数据?

Redis 提供 6 种数据淘汰策略：

1. **volatile-lru（least recently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

7. **volatile-lfu（least frequently used）**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
8. **allkeys-lfu（least frequently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

### LRU算法

##### 什么是LRU?

上面说到了Redis可使用最大内存使用完了，是可以使用LRU算法进行内存淘汰的，那么什么是LRU算法呢？

> **LRU(Least Recently Used)**，即最近最少使用，是一种缓存置换算法。在使用内存作为缓存的时候，缓存的大小一般是固定的。当缓存被占满，这个时候继续往缓存里面添加数据，就需要淘汰一部分老的数据，释放内存空间用来存储新的数据。这个时候就可以使用LRU算法了。其核心思想是：如果一个数据在最近一段时间没有被用到，那么将来被使用到的可能性也很小，所以就可以被淘汰掉。

##### 使用java实现一个简单的LRU算法

```java
 public class LRUCache<k, v> {   
        //容量    
        private int capacity;    
        //当前有多少节点的统计   
        private int count;    
        //缓存节点    
        private Map<k, Node<k, v>> nodeMap;   
        private Node<k, v> head;    
        private Node<k, v> tail;    
        public LRUCache(int capacity) { 
            if (capacity < 1) {  
                throw new IllegalArgumentException(String.valueOf(capacity));        
            }        
            this.capacity = capacity; 
            this.nodeMap = new HashMap<>();   
            //初始化头节点和尾节点，利用哨兵模式减少判断头结点和尾节点为空的代码
            Node headNode = new Node(null, null);    
            Node tailNode = new Node(null, null);      
            headNode.next = tailNode;        
            tailNode.pre = headNode;        
            this.head = headNode;        
            this.tail = tailNode;    
        }    
        public void put(k key, v value) {
            Node<k, v> node = nodeMap.get(key);
        }        if (node == null) {           
            if (count >= capacity) {                
                //先移除一个节点                
                removeNode();           
            }            
            node = new Node<>(key, value);
            //添加节点            
            addNode(node);        
        } else {            
            //移动节点到头节点            
            moveNodeToHead(node);        
        }    
    }    
    public Node<k, v> get(k key) { 
        Node<k, v> node = nodeMap.get(key);
        if (node != null) { 
            moveNodeToHead(node);   
        }        return node;    
    }    private void removeNode() { 
        Node node = tail.pre; 
        //从链表里面移除        
        removeFromList(node);        
        nodeMap.remove(node.key);  
        count--;   
    }    
    private void removeFromList(Node<k, v> node) {
        Node pre = node.pre;
    }       
    Node next = node.next;     
    pre.next = next;        
    next.pre = pre;        
    node.next = null;        
    node.pre = null;    
}    private void addNode(Node<k, v> node) {
    //添加节点到头部        
    addToHead(node);        
    nodeMap.put(node.key, node); 
    count++;   
}    
private void addToHead(Node<k, v> node) {
    Node next = head.next;        
    next.pre = node;        
    node.next = next;        
    node.pre = head;        
    head.next = node;    
}    
public void moveNodeToHead(Node<k, v> node) {
    //从链表里面移除        
    removeFromList(node);       
    //添加节点到头部        
    addToHead(node);    
}    
class Node<k, v> {   
    k key;        
    v value;        
    Node pre;        
    Node next;        
    public Node(k key, v value) {
        this.key = key;            
        this.value = value;        
    }    
}
}
```

> 上面这段代码实现了一个简单的LUR算法，代码很简单，也加了注释，仔细看一下很容易就看懂。

### LRU在Redis中的实现

关于 Redis 中的 LRU 算法，官网上是这样说的：

https://github.com/redis/redis-doc/blob/master/topics/lru-cache.md

![其实吧，LRU也就那么回事](https://p3-tt.byteimg.com/origin/pgc-image/057d050740a94da68fd1b6535385a78d?from=pc)



在 Redis 中的 LRU 算法不是严格的 LRU 算法。

Redis 会尝试执行一个近似的LRU算法，通过采样一小部分键，然后在采样键中回收最适合的那个，也就是最久没有被访问的那个（with the oldest access time）。

然而，从 Redis3.0 开始，改善了算法的性能，使得更接近于真实的 LRU 算法。做法就是维护了一个回收候选键池。

Redis 的 LRU 算法有一个非常重要的点就是你可以通过修改下面这个参数的配置，自己调整算法的精度。

maxmemory-samples 5

最重要的一句话我也已经标志出来了：

The reason why Redis does not use a true LRU implementation is because it costs more memory.

Redis 没有使用真实的 LRU 算法的原因是因为这会消耗更多的内存。

##### 近似LRU算法

Redis使用的是近似LRU算法，它跟常规的LRU算法还不太一样。近似LRU算法通过随机采样法淘汰数据，每次随机出5（默认）个key，从里面淘汰掉最近最少使用的key。

> 可以通过maxmemory-samples参数修改采样数量：例：maxmemory-samples 10 maxmenory-samples配置的越大，淘汰的结果越接近于严格的LRU算法

Redis为了实现近似LRU算法，给每个key增加了一个额外增加了一个24bit的字段，用来存储该key最后一次被访问的时间。

##### Redis3.0对近似LRU的优化

Redis3.0对近似LRU算法进行了一些优化。新算法会维护一个候选池（大小为16），池中的数据根据访问时间进行排序，第一次随机选取的key都会放入池中，随后每次随机选取的key只有在访问时间小于池中最小的时间才会放入池中，直到候选池被放满。当放满后，如果有新的key需要放入，则将池中最后访问时间最大（最近被访问）的移除。

当需要淘汰的时候，则直接从池中选取最近访问时间最小（最久没被访问）的key淘汰掉就行。

##### LRU算法的对比

我们可以通过一个实验对比各LRU算法的准确率，先往Redis里面添加一定数量的数据n，使Redis可用内存用完，再往Redis里面添加n/2的新数据，这个时候就需要淘汰掉一部分的数据，如果按照严格的LRU算法，应该淘汰掉的是最先加入的n/2的数据。生成如下各LRU算法的对比图（[图片来源]

![面试官问：Redis 内存满了怎么办？我想不到！](https://p6-tt.byteimg.com/origin/pgc-image/36f9b74f5e3c41129a4fb2b430624310?from=pc)



你可以看到图中有三种不同颜色的点：

- 浅灰色是被淘汰的数据
- 灰色是没有被淘汰掉的老数据
- 绿色是新加入的数据

我们能看到Redis3.0采样数是10生成的图最接近于严格的LRU。而同样使用5个采样数，Redis3.0也要优于Redis2.8。

### LFU算法

LFU算法是Redis4.0里面新加的一种淘汰策略。它的全称是**Least Frequently Used**，它的核心思想是根据key的最近被访问的频率进行淘汰，很少被访问的优先被淘汰，被访问的多的则被留下来。

LFU算法能更好的表示一个key被访问的热度。假如你使用的是LRU算法，一个key很久没有被访问到，只刚刚是偶尔被访问了一次，那么它就被认为是热点数据，不会被淘汰，而有些key将来是很有可能被访问到的则被淘汰了。如果使用LFU算法则不会出现这种情况，因为使用一次并不会使一个key成为热点数据。

LFU一共有两种策略：

- volatile-lfu：在设置了过期时间的key中使用LFU算法淘汰key
- allkeys-lfu：在所有的key中使用LFU算法淘汰数据

> 设置使用这两种淘汰策略跟前面讲的一样，不过要注意的一点是这两种策略只能在Redis4.0及以上设置，如果在Redis4.0以下设置会报错

**数据库中有 3000w 的数据，而 Redis 中只有 100w 数据，如何保证 Redis 中存放的都是热点数据？**

这个题你说它的考点是什么？

考的就是淘汰策略呀，同志们，只是方式比较隐晦而已。

我们先指定淘汰策略为 allkeys-lru 或者 volatile-lru，然后再计算一下 100w 数据大概占用多少内存，根据算出来的内存，限定 Redis 占用的内存

# **LRU 在 MySQL 中的应用**

LRU 在 MySQL 的应用就是 Buffer Pool，也就是缓冲池。

它的目的是为了减少磁盘 IO。

缓冲池具体是干啥的，我这里就不展开说了。

你就知道它是一块连续的内存，默认大小 128M，可以进行修改。

这一块连续的内存，被划分为若干默认大小为 16KB 的页。

既然它是一个 pool，那么必然有满了的时候，怎么办？

就得移除某些页了，对吧？

那么问题就来了：移除哪些页呢？

刚刚说了，它是为了减少磁盘 IO。所以应该淘汰掉很久没有被访问过的页。

很久没有使用，这不就是 LRU 的主场吗？

但是在 MySQL 里面并不是简单的使用了 LRU 算法。

因为 MySQL 里面有一个预读功能。预读的出发点是好的，但是有可能预读到并不需要被使用的页。

这些页也被放到了链表的头部，容量不够，导致尾部元素被淘汰。

哦豁，降低命中率了，凉凉。

还有一个场景是全表扫描的 sql，有可能直接把整个缓冲池里面的缓冲页都换了一遍，影响其他查询语句在缓冲池的命中率。

那么怎么处理这种场景呢？

把 LRU 链表分为两截，一截里面放的是热数据，一截里面放的是冷数据

# LRU算法实现

# **方案一：数组**

如果之前完全没有接触过 LRU 算法，仅仅知道其思路。

第一次听就要求你给一个实现方案，那么数组的方案应该是最容易想到的。

假设我们有一个定长数组。数组中的元素都有一个标记。这个标记可以是时间戳，也可以是一个自增的数字。

假设我们用自增的数字。

每放入一个元素，就把数组中已经存在的数据的标记更新一下，进行自增。当数组满了后，就将数字最大的元素删除掉。

每访问一个元素，就将被访问的元素的数字置为 0 。

这不就是 LRU 算法的一个实现方案吗？

按照这个思路，撸一份七七八八的代码出来，问题应该不大吧？

**但是这一种方案的弊端也是很明显：需要不停地维护数组中元素的标记。**

那么你觉得它的时间复杂度是多少？

是的，每次操作都伴随着一次遍历数组修改标记的操作，所以时间复杂度是O(n)。

但是这个方案，面试官肯定是不会满意的。因为，这不是他心中的标准答案。

也许他都没想过：你还能给出这种方案呢？

但是它不会说出来，只会轻轻的说一句：还有其他的方案吗？



# **方案二：链表**

最近最少使用，感觉是需要一个有序的结构。

每插入一个元素的时候，就追加在数组的末尾。

每访问一次元素，就把被访问的元素移动到数组的末尾。

这样最近被用的一定是在最后面的，头部的就是最近最少使用的。

当指定长度被用完了之后，就把头部元素移除掉就行了。

这是个什么结构？

这不就是个链表吗？

维护一个有序单链表，越靠近链表头部的结点是越早之前访问的。

当有一个新的数据被访问时，我们从链表头部开始顺序遍历链表。

如果此数据之前已经被缓存在链表中了，我们遍历得到这个数据的对应结点，并将其从原来的位置删除，并插入到链表尾部。

如果此数据没在缓存链表中，怎么办？

分两种情况：

> 如果此时缓存未满，可直接在链表尾部插入新节点存储此数据；
>
> 如果此时缓存已满，则删除链表头部节点，再在链表尾部插入新节点。

**这个方案比数组的方案好在哪里呢？**

从时间复杂度的角度看，因为链表插入、查询的时候都要遍历链表，查看数据是否存在，所以它还是O(n)。

**查询和插入的时间复杂度都是O(1)的解决方案？**

# **方案三：双向链表+哈希表。**

如果想要查询和插入的时间复杂度都是O(1)，那么我们需要一个满足下面三个条件的数据结构：

> 1.首先这个数据结构必须是有时序的，以区分最近使用的和很久没有使用的数据，当容量满了之后，要删除最久未使用的那个元素。
>
> 2.要在这个数据结构中快速找到某个 key 是否存在，并返回其对应的 value。
>
> 3.每次访问这个数据结构中的某个 key，需要将这个元素变为最近使用的。也就是说，这个数据结构要支持在任意位置快速插入和删除元素。

那么，什么样的数据结构同时符合上面的条件呢？

查找快，哈希表。但是哈希表的数据是乱序的。

有序，链表，插入、删除都很快，但是查询慢。

**所以，得让哈希表和链表结合一下，成长一下，形成一个新的数据结构，那就是：哈希链表，LinkedHashMap。**

这个结构大概长这样：

![其实吧，LRU也就那么回事](https://p6-tt.byteimg.com/origin/pgc-image/d725c99338a54980afa296a9dbff26b8?from=pc)



借助这个结构，我们再来分析一下上面的三个条件：

> 1.如果每次默认从链表尾部添加元素，那么显然越靠近尾部的元素就越是最近使用的。越靠近头部的元素就是越久未使用的。
>
> 2.对于某一个 key ，可以通过哈希表快速定位到链表中的节点，从而取得对应的 value。
>
> 3.链表显然是支持在任意位置快速插入和删除的，修改指针就行。但是单链表无法按照索引快速访问某一个位置的元素，都是需要遍历链表的，所以这里借助哈希表，可以通过 key，快速的映射到任意一个链表节点，然后进行插入和删除。

**为什么这里要用双链表呢，单链表为什么不行？**

涉及到删除元素的操作

那么链表删除元素除了自己本身的指针信息，还需要什么东西？

是不是还需要前驱节点的指针？

那么我们这里要求时间复杂度是O(1)，所以怎么才能直接获取到前驱节点的指针？

这玩意是不是就得上双链表？

**哈希表里面已经保存了 key ，那么链表中为什么还要存储 key 和 value 呢，只存入 value 不就行了？**

刚刚删除链表中的节点，需要借助双链表来实现O(1)。

删除了链表中的节点，然后呢？

是不是还忘记了什么东西？

是不是还有一个哈希表忘记操作了？

哈希表是不是也得进行对应的删除操作？

删除哈希表需要什么东西？

是不是需要 key，才能删除对应的 value？

这个 key 从哪里来？

是不是只能从链表中的结点里面来？

如果链表中的结点，只有 value 没有 key，那么我们就无法删除哈希表的 key。那不就完犊子了吗？
