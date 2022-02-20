## Redis常见对象类型的底层数据结构

Redis 是一个基于内存中的数据结构存储系统，可以用作数据库、缓存和消息中间件。Redis 支持五种常见对象类型：字符串（String）、哈希（Hash）、列表（List）、集合（Set）以及有序集合（Zset）

> Redis 除了这 5 种数据类型之外，还有 Bitmaps、HyperLogLogs、Streams 等。
>
> 本文主要内容参考自《Redis设计与实现》

## String

1. **介绍** ：string 数据结构是简单的 key-value 类型。虽然 Redis 是用 C 语言写的，但是 Redis 并没有使用 C 的字符串表示，而是自己构建了一种 **简单动态字符串**（simple dynamic string，**SDS**）。相比于 C 的原生字符串，Redis 的 SDS 不光可以保存文本数据还可以保存二进制数据，并且获取字符串长度复杂度为 O(1)（C 字符串为 O(N)）,除此之外,Redis 的 SDS API 是安全的，不会造成缓冲区溢出。
2. **常用命令:** `set,get,strlen,exists,decr,incr,setex` 等等。
3. **应用场景** ：一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞转发数量等等。

## list

1. **介绍** ：**list** 即是 **链表**。链表是一种非常常见的数据结构，特点是易于数据元素的插入和删除并且且可以灵活调整链表长度，但是链表的随机访问困难。许多高级编程语言都内置了链表的实现比如 Java 中的 **LinkedList**，但是 C 语言并没有实现链表，所以 Redis 实现了自己的链表数据结构。Redis 的 list 的实现为一个 **双向链表**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。
2. **常用命令:** `rpush,lpop,lpush,rpop,lrange、llen` 等。
3. **应用场景:** 发布与订阅或者说消息队列、慢查询、比如可以通过 lrange 命令，读取某个闭区间内的元素，可以基于 list 实现分页查询，这个是很棒的一个功能，基于 Redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。。

##  hash

1. **介绍** ：hash 类似于 JDK1.8 前的 HashMap，内部实现也差不多(数组 + 链表)。不过，Redis 的 hash 做了更多优化。另外，hash 是一个 string 类型的 field 和 value 的映射表，**特别适合用于存储对象**，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以 hash 数据结构来存储用户信息，商品信息等等。
2. **常用命令：** `hset,hmset,hexists,hget,hgetall,hkeys,hvals` 等。
3. **应用场景:** 系统中对象数据的存储。

## set

1. **介绍 ：** set 类似于 Java 中的 `HashSet` 。Redis 中的 set 类型是一种无序集合，集合中的元素没有先后顺序。当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。可以基于 set 轻易实现交集、并集、差集的操作。比如：你可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程。
2. **常用命令：** `sadd,spop,smembers,sismember,scard,sinterstore,sunion` 等。
3. **应用场景:** 需要存放的数据不能重复以及需要获取多个数据源交集和并集等场景

## sorted set/Zset

1. **介绍：** 和 set 相比，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，还可以通过 score 的范围来获取元素的列表。有点像是 Java 中 HashMap 和 TreeSet 的结合体。
2. **常用命令：** `zadd,zcard,zscore,zrange,zrevrange,zrem` 等。
3. **应用场景：** 需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息。

## bitmap

1. **介绍 ：** bitmap 存储的是连续的二进制数字（0 和 1），通过 bitmap, 只需要一个 bit 位来表示某个元素对应的值或者状态，key 就是对应元素本身 。我们知道 8 个 bit 可以组成一个 byte，所以 bitmap 本身会极大的节省储存空间。
2. **常用命令：** `setbit` 、`getbit` 、`bitcount`、`bitop`
3. **应用场景:** 适合需要保存状态信息（比如是否签到、是否登录...）并需要进一步对这些信息进行分析的场景。比如用户签到情况、活跃用户情况、用户行为统计（比如是否点赞过某个视频）。

## **1. 对象类型和编码**

Redis 使用对象来存储键和值的，在Redis中，每个对象都由 redisObject 结构表示。redisObject 结构主要包含三个属性：type、encoding 和 ptr。

```c
typedef struct redisObject {  
// 类型    unsigned type:4;    
// 编码    unsigned encoding:4;    
// 底层数据结构的指针    
void *ptr;
} 
robj;
```

其中 type 属性记录了对象的类型。对于 Redis 来说，键对象总是字符串类型，值对象可以是任意支持的类型。因此，当我们说 Redis 键采用哪种对象类型的时候，指的是对应的值采用哪种对象类型。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCiarZzD3NiatvHAIfNlJsFkicvkwG7aLlrA20TAt7QXodgMxuTcZCZv1sA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



*ptr 属性指向了对象的底层数据结构，而这些数据结构由 encoding 属性决定。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCDvaicE0icwehy766VfBGh7mVnIDuicZ00trRFtlZicIuqLw2lef1CIKUZg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

之所以由 encoding 属性来决定对象的底层数据结构，是为了实现同一对象类型，支持不同的底层实现。这样就能在不同场景下，使用不同的底层数据结构，进而极大提升Redis的灵活性和效率。

## **2. 字符串对象**

字符串是我们日常工作中用得最多的对象类型，它对应的编码可以是 int、raw 和 embstr。字符串对象相关命令可参考：Redis命令-Strings。

如果一个字符串对象保存的是不超过 long 类型的整数值，此时编码类型即为 int，其底层数据结构直接就是 long 类型。例如执行 set number 10086，就会创建 int 编码的字符串对象作为 number 键的值。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCoDjia9UEdeX3z22wjSy7D7pTgI0PpNgdQK2tYeeiagiayC9QibsDzdXxKQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果字符串对象保存的是一个长度大于 39 字节的字符串，此时编码类型即为 raw，其底层数据结构是简单动态字符串（SDS）；如果长度小于等于 39 个字节，编码类型则为 embstr，底层数据结构就是 embstr 编码 SDS。下面，我们详细理解下什么是简单动态字符串。

### **2.1 简单动态字符串**

#### **SDS 定义**

在 Redis 中，使用 sdshdr 数据结构表示 SDS：

```c
struct sdshdr {  
// 字符串长度    int len;   
// buf数组中未使用的字节数    int free;    
// 字节数组，用于保存字符串    char buf[];
};
```

SDS 遵循了 C 字符串以空字符结尾的惯例，保存空字符的 1 字节不会计算在 len 属性里面。例如，Redis 这个字符串在 SDS 里面的数据可能是如下形式：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCCxHywxeVBc8rJuVicDjZHSqrzSiadjiaIPtYL2w7T50B4pzJYMQc2nszQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**SDS 与 C 字符串的区别**

C 语言使用长度为 N+1 的字符数组来表示长度为N的字符串，并且字符串的最后一个元素是空字符 \0。Redis 采用 SDS 相对于 C 字符串有如下几个优势：

> 1. 常数复杂度获取字符串长度；
> 2. 杜绝缓冲区溢出；
> 3. 减少修改字符串时带来的内存重分配次数；
> 4. 二进制安全。

**常数复杂度获取字符串长度**

因为 C 字符串并不记录自身的长度信息，所以为了获取字符串的长度，必须遍历整个字符串，时间复杂度是 **O(N**)。而 SDS 使用 len 属性记录了字符串的长度，因此获取 SDS字符串长度的时间复杂度是 **O(1)**。

##### **杜绝缓冲区溢出**

C 字符串不记录自身长度带来的另一个问题是，**很容易造成缓存区溢出**。比如使用字符串拼接函数（stract）的时候，很容易覆盖掉字符数组原有的数据。

与 C 字符串不同，SDS 的空间分配策略完全杜绝了发生缓存区溢出的可能性。当 SDS 进行字符串扩充时，首先会检查当前的字节数组的长度是否足够。如果不够的话，会先进行自动扩容，然后再进行字符串操作。

##### **减少修改字符串时带来的内存重分配次数**

因为 C 字符串的长度和底层数据是紧密关联的，所以每次增长或者缩短一个字符串，程序都要对这个数组进行一次内存重分配：

> - 如果是增长字符串操作，需要先通过内存重分配来扩展底层数组空间大小，不这么做就导致缓存区溢出；
> - 如果是缩短字符串操作，需要先通过内存重分配来来回收不再使用的空间，不这么做就导致内存泄漏。

因为内存重分配涉及复杂的算法，并且可能需要执行系统调用，所以通常是个比较耗时的操作。对于 Redis 来说，字符串修改是一个十分频繁的操作。如果每次都像 C 字符串那样进行内存重分配，对性能影响太大了，显然是无法接受的。

SDS 通过空闲空间解除了字符串长度和底层数据之间的关联。在 SDS 中，数组中可以包含未使用的字节，这些字节数量由 free 属性记录。**通过空闲空间，SDS 实现了空间预分配和惰性空间释放两种优化策略**。

*1. 空间预分配*

空间预分配是用于优化 SDS 字符串增长操作的，简单来说就是当字节数组空间不足触发重分配的时候，总是会预留一部分空闲空间。这样的话，就能减少连续执行字符串增长操作时的内存重分配次数。

有两种预分配的策略：

> - len 小于 1MB 时：每次重分配时会多分配同样大小的空闲空间；
> - len 大于等于 1MB 时：每次重分配时会多分配 1MB 大小的空闲空间。

*2. 惰性空间释放*

惰性空间释放是用于优化 SDS 字符串缩短操作的。简单来说就是当字符串缩短时，并不立即使用内存重分配来回收多出来的字节，而是用 free 属性记录，等待将来使用。SDS 也提供直接释放未使用空间的 API，在需要的时候，也能真正的释放掉多余的空间。

##### **二进制安全**

C 字符串中的字符必须符合某种编码，并且除了字符串末尾之外，其它位置不允许出现空字符。这些限制使得 C 字符串只能保存文本数据。

但是对于 Redis 来说，不仅仅需要保存文本，还要支持保存二进制数据。为了实现这一目标，SDS 的 API 全部做到了二进制安全（binary-safe）。

### **2.2 raw 和 embstr 编码的 SDS 区别**

我们在前面讲过，长度大于 39 字节的字符串，编码类型为 raw，底层数据结构是简单动态字符串（SDS）。这个很好理解，比如当我们执行 set story "Long, long, long ago there lived a king ..."（长度大于39）之后，Redis 就会创建一个 raw 编码的 String 对象。

数据结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCGYoeXic4QPsN9v944eMaicj96zC1Mficw106gBEGMib13KofZFjgeJYtsQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

长度小于等于 39 个字节的字符串，编码类型为 embstr，底层数据结构则是 embstr 编码 SDS。embstr 编码是专门用来保存短字符串的，它和 raw 编码最大的不同在于：raw 编码会调用两次内存分配分别创建 redisObject 结构和 sdshdr 结构；而 embstr 编码则是只调用一次内存分配，在一块连续的空间上同时包含 redisObject 结构和 sdshdr 结构。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCHUoeEClIAQ2LU2lba3lSnqwZ9iaIGnjaEFaK5sFCuHibav62jSSJJqgQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### **2.3 编码转换**

int 编码和 embstr 编码的字符串对象在条件满足的情况下会自动转换为 raw 编码的字符串对象。

对于 int 编码来说，当我们修改这个字符串为不再是整数值的时候，此时字符串对象的编码就会从 int 变为 raw。

对于 embstr 编码来说，只要我们修改了字符串的值，此时字符串对象的编码就会从 embstr 变为 raw。

embstr 编码的字符串对象可以认为是只读的，因为 Redis 为其编写任何修改程序。当我们要修改 embstr 编码字符串时，都是先将转换为 raw 编码，然后再进行修改。

## **3. 列表对象**

列表对象的编码可以是 linkedlist 或者 ziplist，对应的底层数据结构是链表和压缩列表。列表对象相关命令可参考：Redis 命令-List。

默认情况下，当列表对象保存的所有字符串元素的长度都小于 64 字节，且元素个数小于 512 个时，列表对象采用的是 ziplist 编码，否则使用 linkedlist 编码。

可以通过配置文件修改该上限值。

### **3.1 链表**

链表是一种非常常见的数据结构，提供了高效的节点重排能力以及顺序性的节点访问方式。在 Redis 中，每个链表节点使用 listNode 结构表示：

```c
typedef struct listNode {    
// 前置节点    struct listNode *prev;    
// 后置节点    struct listNode *next;   
// 节点值    void *value;
} 
listNode
```



多个 listNode 通过 prev 和 next 指针组成双端链表，如下图所示：

![image-20210712163131010](/Users/tianxiaofeng/Library/Application Support/typora-user-images/image-20210712163131010.png)

为了操作起来比较方便，Redis 使用了 list 结构持有链表。

```c
typedef struct list {    
// 表头节点    listNode *head;    
// 表尾节点    listNode *tail;   
// 链表包含的节点数量    unsigned long len;    
// 节点复制函数    void *(*dup)(void *ptr);   
// 节点释放函数    void (*free)(void *ptr);    
// 节点对比函数    int (*match)(void *ptr, void *key);
} 
list;
```

list 结构为链表提供了表头指针 head、表尾指针 tail，以及链表长度计数器 len，而 dup、free 和 match 成员则是实现多态链表所需类型的特定函数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCd4EFfTZXqJNojX9rUHQJVtSma1FHI5M2XdOeVibXKxacABXv6eD0ckQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



Redis 链表实现的特征总结如下：

> 1. **双端**：链表节点带有 prev 和 next 指针，获取某个节点的前置节点和后置节点的复杂度都是 **O(n)**；
> 2. **无环**：表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL，对链表的访问以 NULL 为终点；
> 3. **带表头指针和表尾指针**：通过 list 结构的 head 指针和 tail 指针，程序获取链表的表头节点和表尾节点的复杂度为 **O(1)**；
> 4. **带链表长度计数器**：程序使用 list 结构的 len 属性来对 list 持有的节点进行计数，程序获取链表中节点数量的复杂度为 **O(1)**；
> 5. **多态**：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。

### **3.2 压缩列表**

压缩列表（ziplist）是列表键和哈希键的底层实现之一。压缩列表主要目的是为了节约内存，是由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCOkibB6TsXyDWyyHicez7pm7Sktvebx4s8YibIwqBosQgYibYPibqyOanm0w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，压缩列表记录了各组成部分的类型、长度以及用途。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCpicAZEibjWqZ5xZf6iapLKF2xqR3cTbtLfbF26OF2y9ZVKD2RhzMCD5tw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **4. 哈希对象**

哈希对象的编码可以是 ziplist 或者 hashtable。

### **4.1 hash-ziplist**

ziplist 底层使用的是压缩列表实现，上文已经详细介绍了压缩列表的实现原理。每当有新的键值对要加入哈希对象时，先把保存了键的节点推入压缩列表表尾，然后再将保存了值的节点推入压缩列表表尾。比如，我们执行如下三条 HSET 命令：

```c
HSET profile name "tom"HSET profile age 25HSET profile career "Programmer"
```

如果此时使用 ziplist 编码，那么该 Hash 对象在内存中的结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCv8G8OoibfPe8TngkbEhcLrUsG3jx8QibPZ1bMYY4bD91kHVCGKd6bkBA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### **3.2 hash-hashtable**

hashtable 编码的哈希对象使用字典作为底层实现。字典是一种用于保存键值对的数据结构，Redis 的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，每个哈希表节点保存的就是一个键值对。

#### **3.3 哈希表**

Redis 使用的哈希表由 dictht 结构定义：

```c
typedef struct dictht{    
		// 哈希表数组    dictEntry **table;
 		// 哈希表大小    unsigned long size;
    // 哈希表大小掩码，用于计算索引值    // 总是等于 size-1    unsigned long sizemask;
    // 该哈希表已有节点数量    unsigned long used;
    }
    dictht
```

table 属性是一个数组，数组中的每个元素都是一个指向 dictEntry 结构的指针，每个 dictEntry 结构保存着一个键值对。

size 属性记录了哈希表的大小，即 table 数组的大小。used 属性记录了哈希表目前已有节点数量。sizemask 总是等于 size-1，这个值主要用于数组索引。

比如下图展示了一个大小为 4 的空哈希表。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCxhS5wAuC1NibJa4lXuPV3vWPEHJobYNXTW8Z7OQJ9YicnBULlC7sxmug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### **哈希表节点**

哈希表节点使用 dictEntry 结构表示，每个 dictEntry 结构都保存着一个键值对：

```c
typedef struct dictEntry {    
		// 键    void *key;
    // 值    union {        void *val;       
    												unit64_t u64;       
                            nit64_t s64;    
                            }
                            v;
    // 指向下一个哈希表节点，形成链表    struct dictEntry *next;
    } 
    dictEntry;
```

key 属性保存着键值对中的键，而 v 属性则保存了键值对中的值。值可以是一个指针，一个 uint64_t 整数或者是 int64_t 整数。next 属性指向了另一个 dictEntry 节点，在数组桶位相同的情况下，将多个 dictEntry 节点串联成一个链表，以此来解决键冲突问题（链地址法）。

#### **3.4 字典**

Redis 字典由 dict 结构表示：

```c
typedef struct dict {    
		// 类型特定函数    dictType *type;
    // 私有数据    void *privdata;
    // 哈希表    dictht ht[2];
    //rehash索引    
    // 当rehash不在进行时，值为-1    
    int rehashidx;
    }
```

ht 是大小为 2，且每个元素都指向 dictht 哈希表。一般情况下，字典只会使用 ht[0] 哈希表，ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。rehashidx 记录了 rehash 的进度，如果目前没有进行 rehash，值为 -1。

#### **rehash**

为了使 hash 表的负载因子 (ht[0]).used/ht[0]).size) 维持在一个合理范围，当哈希表保存的元素过多或者过少时，程序需要对 hash 表进行相应的扩展和收缩。

rehash（重新散列）操作就是用来完成 hash 表的扩展和收缩的。

rehash 的步骤如下：

> 1. 为 ht [1] 哈希表分配空间；
>
> - 如果是**扩展操作**，那么 ht[1] 的大小为第一个大于 ht[0].used*2的2n。比如ht[0].used=5，那么此时 ht[1] 的大小就为 16（大于 10 的第一个 2n 的值是 16）；
> - 如果是**收缩操作**，那么 ht[1] 的大小为第一个大于 ht[0].used 的 2n。比如ht[0].used=5，那么此时 ht[1] 的大小就为 8（大于 5 的第一个 2n 的值是 8）。
>
> 2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 中；
>
> 3. 迁移完成之后，释放掉 ht[0]，并将现在的 ht[1] 设置为 ht[0]，在 ht[1] 新创建一个空白哈希表，为下一次 rehash 做准备。

**哈希表的扩展和收缩时机**

- 当服务器没有执行 BGSAVE 或者 BGREWRITEAOF 命令时，负载因子大于等于 1 触发哈希表的扩展操作；
- 当服务器在执行 BGSAVE 或者 BGREWRITEAOF 命令，负载因子大于等于 5 触发哈希表的扩展操作；
- 当哈希表负载因子小于 0.1，触发哈希表的收缩操作。

#### **渐进式 rehash**

前面讲过，扩展或者收缩需要将 ht[0] 里面的元素全部 rehash 到 ht[1] 中，如果 ht[0] 元素很多，显然一次性 rehash 成本会很大，从影响到 Redis 性能。

为了解决上述问题，Redis 使用了渐进式 rehash 技术，具体来说就是分多次，渐进式地将 ht[0] 里面的元素慢慢地 rehash 到 ht[1] 中。

下面是渐进式 rehash 的详细步骤：

> 1. 为 ht[1] 分配空间；
> 2. 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为 0，表示 rehash 正式开始；
> 3. 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新时，除了会执行相应的操作之外，还会顺带将 ht[0] 在 rehashidx 索引位上的所有键值对 rehash 到 ht[1] 中，rehash 完成之后，rehashidx 值加 1；
> 4. 随着字典操作的不断进行，最终会在啊某个时刻迁移完成，此时将 rehashidx 值置为 -1，表示 rehash 结束。

**渐进式 rehash 一次迁移一个桶上所有的数据**。设计上采用**分而治之**的思想，**将原本集中式的操作分散到每个添加、删除、查找和更新操作上**，从而避免集中式 rehash 带来的庞大计算。

因为在渐进式 rehash 时，字典会同时使用 ht[0] 和 ht[1] 两张表，所以此时对字典的删除、查找和更新操作都可能会在两个哈希表进行。比如，如果要查找某个键时，先在 ht[0] 中查找，如果没找到，则继续到 ht[1] 中查找。

#### **hash 对象中的 hashtable**

```c
HSET profile name "tom"HSET profile age 25HSET profile career "Programmer"
```

还是上述三条命令，保存数据到 Redis 的哈希对象中，如果采用 hashtable 编码保存的话，那么该 Hash 对象在内存中的结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCJUxJVgMD1AgfZZyHgtsxdV3G1aSibKkNw1zIFfqI1IcichydWGxQib3zg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当哈希对象保存的所有键值对的键和值的字符串长度都小于 64 个字节，并且数量小于 512 个时，使用 ziplist 编码，否则使用 hashtable 编码。

可以通过配置文件修改该上限值。

## **4. 集合对象**

集合对象的编码可以是 intset 或者 hashtable。当集合对象保存的元素都是整数，并且个数不超过 512 个时，使用 intset 编码，否则使用 hashtable 编码。

### **4.1 set-intset**

intset 编码的集合对象底层使用整数集合实现。

整数集合（intset）是 Redis 用于保存整数值的集合抽象数据结构，它可以保存类型为 int16_t、int32_t 或者 int64_t 的整数值，并且保证集合中的数据不会重复。Redis 使用 intset 结构表示一个整数集合。

```c
typedef struct intset {    
// 编码方式    uint32_t encoding;    
// 集合包含的元素数量    uint32_t length;    
// 保存元素的数组    int8_t contents[];
} 
intset;
```

contents 数组是整数集合的底层实现：整数集合的每个元素都是 contents 数组的一个数组项，各个项在数组中按值大小从小到大有序排列，并且数组中不包含重复项。

虽然 contents 属性声明为 int8_t 类型的数组，但实际上，contents 数组不保存任何 int8_t 类型的值，数组中真正保存的值类型取决于 encoding。

如果 encoding 属性值为 INTSET_ENC_INT16，那么 contents 数组就是 int16_t 类型的数组，以此类推。

当新插入元素的类型比整数集合现有类型元素的类型大时，整数集合必须先升级，然后才能将新元素添加进来。这个过程分以下三步进行：

> 1. 根据新元素类型，扩展整数集合底层数组空间大小；
> 2. 将底层数组现有所有元素都转换为与新元素相同的类型，并且维持底层数组的有序性；
> 3. 将新元素添加到底层数组里面。

还有一点需要注意的是，**整数集合不支持降级**。一旦对数组进行了升级，编码就会一直保持升级后的状态。

举个例子，当执行 SADD numbers 1 3 5 向集合对象插入数据时，该集合对象在内存的结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCJCILdbRWe9piaYNCpl8e5pSGSuz1jpWeaSexl0ZicgkTxRiaICrYkicY2A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### **4.2 set-hashtable**

hashtable 编码的集合对象使用字典作为底层实现。字典的每个键都是一个字符串对象，每个字符串对象对应一个集合元素，字典的值都是 NULL。

当我们执行 SADD fruits "apple" "banana" "cherry" 向集合对象插入数据时，该集合对象在内存的结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCWrxutMaqrOmUELlz0h4zrwVicZ7kbwzy6obu38op6kDenBRgcI7vXCA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **5. 有序集合对象**

有序集合的编码可以是 ziplist 或者 skiplist。当有序集合保存的元素个数小于 128 个，且所有元素成员长度都小于 64 字节时，使用 ziplist 编码，否则使用 skiplist 编码。

### **5.1 zset-ziplist**

ziplist 编码的有序集合使用压缩列表作为底层实现。每个集合元素使用两个紧挨着一起的两个压缩列表节点表示，第一个节点保存元素的成员（member），第二个节点保存元素的分值（score）。

压缩列表内的集合元素按照分值从小到大排列。如果我们执行 ZADD price 8.5 apple 5.0 banana 6.0 cherry 命令向有序集合插入元素，该有序集合在内存中的结构如下：

###  ![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYC6coByeFo8nOuFZIHSEiadxPzVe3j0P1c6uoLd2qUqGmxdLibLX5UpAgg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### **5.2 zset-skiplist**

skiplist 编码的有序集合对象使用 zset 结构作为底层实现，一个 zset 结构同时包含一个字典和一个跳跃表。

```c
typedef struct zset {   
  zskiplist *zs1;   
  dict *dict;
}
```

继续介绍之前，我们先了解一下什么是跳跃表。

#### **跳跃表**

跳跃表（skiplist）是一种有序的数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

Redis 的跳跃表由 zskiplistNode 和 zskiplist 两个结构定义。zskiplistNode 结构表示跳跃表节点，zskiplist 保存跳跃表节点相关信息，比如节点的数量，以及指向表头和表尾节点的指针等。

##### **跳跃表节点 zskiplistNode**

跳跃表节点 zskiplistNode 结构定义如下：

```c
typedef struct zskiplistNode {    
// 后退指针    struct zskiplistNode *backward;    
// 分值    double score;   
// 成员对象    robj *obj;    
// 层    struct zskiplistLevel {        
// 前进指针        struct zskiplistNode *forward;        
// 跨度        unsigned int span;   
}
level[];
} 
zskiplistNode;
```



下图是一个层高为 5，包含 4 个跳跃表节点（1 个表头节点和 3 个数据节点）组成的跳跃表：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYCytsOU7QQwIfS4BpyFK2XuTHib4x9s1qkUWJgfBrxUQcQQFVkrurBJ3Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### **有序集合对象的 skiplist 实现**

前面讲过，skiplist 编码的有序集合对象使用 zset 结构作为底层实现。一个 zset 结构同时包含一个字典和一个跳跃表。

```c
typedef struct zset {    zskiplist *zs1;    dict *dict;}
```

zset 结构中的 zs1 跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素。

通过跳跃表，可以对有序集合进行基于 score 的快速范围查找。zset 结构中的 dict 字典为有序集合创建了从成员到分值的映射，字典的键保存了成员，字典的值保存了分值。通过字典，可以用 **O(1)** 复杂度查找给定成员的分值。

假如还是执行 ZADD price 8.5 apple 5.0 banana 6.0 cherry 命令向 zset 保存数据，如果采用 skiplist 编码方式的话，该有序集合在内存中的结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxcen4vx8QkeqV8WGFkmqYC5ybxGWxjsjfb4dJejtDSicHb12bgkSCibB7x8VRG9b4sujcvyTeXt7wQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## **6. 总结**

总的来说，Redis 底层数据结构主要包括简单动态字符串（SDS）、链表、字典、跳跃表、整数集合和压缩列表六种类型。并且基于这些基础数据结构实现了字符串对象、列表对象、哈希对象、集合对象以及有序集合对象五种常见的对象类型。每一种对象类型都至少采用了 2 种数据编码，不同的编码使用的底层数据结构也不同。