# join算法

>  **Nested Loop Join (循环嵌套连接)**
>
> - Nested Loop Join策略
> - Index Nested-Loop Join 策略
> - Block Nested-Loop Join策略
>
> **Hash Join(散列连接) **
>
> **Sort Merge Join(排序归并连接)**

## **Nested Loop Join (循环嵌套连接)**

### 	Nested Loop Join 

​	Nested Loop Join 是扫描驱动表，每读出一条记录，就根据 join 的关联字段上的索引去被驱动表中查询对应数据。它适用于被连接的数据子集较小的场景，它也是 MySQL join 的**==唯一==**算法实现

### **Index Nested-Loop Join 算法**

​	这种算法在链接查询的时候，驱动表会根据关联字段的索引进行查找，当在索引上找到了符合的值，再回表进行查询，也就是只有当匹配到索引以后才会进行回表。至于驱动表的选择，MySQL优化器一般情况下是会选择记录数少的作为驱动表，但是当SQL特别复杂的时候不排除会出现错误选择。

​	在索引嵌套链接的方式下，如果非驱动表的关联键是主键的话，这样来说性能就会非常的高，如果不是主键的话，关联起来如果返回的行数很多的话，效率就会特别的低，因为要多次的回表操作。先关联索引，然后根据二级索引的主键ID进行回表的操作。这样来说的话性能相对就会很差。

![img](https://pics3.baidu.com/feed/10dfa9ec8a136327b7a9182f457f33eb09fac75b.jpeg?token=e81b5902b0f646129f6476d3c9811355)

### **Block Nested-Loop Join**

​	Block Nested-Loop Join的算法，简称 BNL，它是 MySQL 在被驱动表上无可用索引时使用的 join 算法，其具体流程如下所示：

> - 把表 t2 的数据读取当前线程的 join_buffer 中，t2 上跟条件过滤放入内存中；
> - 扫描表 t1，每取出一行数据，就跟 join_buffer 中的数据进行对比，满足 join 条件的，则放入结果集。

![img](https://pics4.baidu.com/feed/2e2eb9389b504fc2c99a8632362d741691ef6d33.jpeg?token=a54c97e3fdfa3f8f4b955c8747954b74)

## **Hash Join 算法**

​	Hash Join 是扫描驱动表，利用 join 的关联字段在内存中建立散列表，然后扫描被驱动表，每读出一行数据，并从散列表中找到与之对应数据。它是大数据集连接操时的常用方式，适用于驱动表的数据量较小，可以放入内存的场景，它对于**没有索引的大表**和并行查询的场景下能够提供最好的性能。可惜它只适用于等值连接的场景，比如 on a.id = where b.a_id。

![img](https://pics5.baidu.com/feed/21a4462309f790520f884f1bda0344cd7acbd57c.jpeg?token=66156b89db5dbd25bd214c48fb7bc0b5)

**该算法和 Block Nested-Loop Join 有类似之处，只不过是将无序的 Join Buffer 改为了散列表 hash table，从而让数据匹配不再需要将 join buffer 中的数据全部遍历一遍，而是直接通过 hash，以接近 O(1) 的时间复杂度获得匹配的行**，这极大地提高了两张表的 join 速度

## **Sorted Merge Join 算法**

​	Sort Merge Join 则是先根据 join 的关联字段将两张表排序(如果已经排序好了，比如字段上有索引则不需要再排序)，然后在对两张表进行一次归并操作。如果两表已经被排过序，在执行排序合并连接时不需要再排序了，这时Merge Join的性能会优于Hash Join。Merge Join可适于于非等值Join（>，<，>=，<=，但是不包含!=，也即<>）

​	需要注意的是，如果连接的字段已经有索引，也就说已经排好序的话，可以直接进行归并操作，但是如果连接的字段没有索引的话，则它的执行过程如下图所示

![img](https://pics7.baidu.com/feed/55e736d12f2eb938147bc3c600921632e4dd6fec.jpeg?token=9807a337d964a1af821ed38bcf0204f8)

## 区别

![img](https://pics1.baidu.com/feed/37d3d539b6003af356a66ec8e3da555b1138b676.jpeg?token=b57a692f9527b67d1b575951ac209497)

## 优化

1、永远用小结果集驱动大结果集(其本质就是减少外层循环的数据数量)

2、为匹配的条件增加索引(减少内层表的循环匹配次数)

3、增大join buffer size的大小（一次缓存的数据越多，那么内层包的扫表次数就越少）

4、减少不必要的字段查询（字段越少，join buffer 所缓存的数据就越多）