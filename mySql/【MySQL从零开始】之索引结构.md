## **索引的优缺点**

## **优点**

- 索引大大减小了服务器需要扫描的数据量

- 索引可以帮助服务器避免排序和临时表

- 索引可以将随机IO变成顺序IO

- 索引对于InnoDB（对索引支持行级锁）非常重要，因为它可以让查询锁更少的元组。在MySQL5.1和更新的版本中，InnoDB可以在服务器端过滤掉行后就释放锁，但在早期的MySQL版本中，InnoDB直到事务提交时才会解锁。对不需要的元组的加锁，会增加锁的开销，降低并发性。 InnoDB仅对需要访问的元组加锁，而索引能够减少InnoDB访问的元组数。但是只有在存储引擎层过滤掉那些不需要的数据才能达到这种目的。一旦索引不允许InnoDB那样做（即索引达不到过滤的目的），MySQL服务器只能对InnoDB返回的数据进行WHERE操作，此时，已经无法避免对那些元组加锁了。如果查询不能使用索引，MySQL会进行全表扫描，并锁住每一个元组，不管是否真正需要。

- - 关于InnoDB、索引和锁：InnoDB在二级索引上使用共享锁（读锁），但访问主键索引需要排他锁（写锁）

## **缺点**

- 虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。因为更新表时，MySQL不仅要保存数据，还要保存索引文件。
- 建立索引会占用磁盘空间的索引文件。一般情况这个问题不太严重，但如果你在一个大表上创建了多种组合索引，索引文件的会膨胀很快。
- 如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果。
- 对于非常小的表，大部分情况下简单的全表扫描更高效；

索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间研究建立最优秀的索引，或优化查询语句。

因此应该只为最经常查询和最经常排序的数据列建立索引。

MySQL里同一个数据表里的索引总数限制为16个。

## 索引存储类型

> B-TREE 
>
> B+TREE 
>
> HASH 

## B-Tree索引

B-树就是B树，多路搜索树，树高一层意味着多一次的磁盘I/O，下图是3阶B树

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021013023233065.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

### B树的特征：

> - 关键字集合分布在整颗树中；
> - 任何一个关键字出现且只出现在一个结点中；
> - 搜索有可能在非叶子结点结束；
> - 其搜索性能等价于在关键字全集内做一次二分查找；
> - 自动层次控制；

## B+TREE索引

B+树是B-树的变体，也是一种多路搜索树

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130232533425.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

### B+树的特征：

> 所有关键字都出现在叶子结点的链表中（稠密索引），且链表中的关键字恰好是有序的；
> 不可能在非叶子结点命中；
> 非叶子结点相当于是叶子结点的索引（稀疏索引），叶子结点相当于是存储（关键字）数据的数据层；
> 每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历。
> 更适合文件索引系统；

## HASH索引

哈希索引基于哈希表实现，只有精确索引所有列的查询才有效。

对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码。哈希索引将所有的哈希码存储在索引中，同时在哈希表中保存指向每个数据的指针。

MySQL中，只有Memory存储引擎显示支持hash索引，是Memory表的默认索引类型，尽管Memory表也可以使用B-Tree索引。

Memory存储引擎支持非唯一hash索引，这在数据库领域是罕见的：如果多个值有相同的hash code，索引把它们的行指针用链表保存到同一个hash表项中。

哈希索引就是采用一定的哈希算法，把键值换算成新的哈希值，检索时不需要类似B+树那样从根节点到叶子节点逐级查找，只需一次哈希算法即可立刻定位到相应的位置，速度非常快。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021013023265936.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

Hash索引仅仅能满足"=",“IN"和”<=>"查询，不能使用范围查询。也不支持任何范围查询，例如WHERE price > 100。
　　
由于Hash索引比较的是进行Hash运算之后的Hash值，所以它只能用于等值的过滤，不能用于基于范围的过滤，因为经过相应的Hash算法处理之后的Hash值的大小关系，并不能保证和Hash运算前完全一样。

假设创建如下一个表：

```MySQL
CREATE TABLE testhash (
  fname VARCHAR(50) NOT NULL,
  lname VARCHAR(50) NOT NULL,
  KEY USING HASH(fname)
) ENGINE=MEMORY;
```

包含的数据如下：

![img](https://pic4.zhimg.com/80/v2-c57ecf890c1e4acbe53560592df2a6e3_1440w.jpg)

假设索引使用hash函数f( )，如下：

```MySQL
f('Arjen') = 2323
f('Baron') = 7437
f('Peter') = 8784
f('Vadim') = 2458
```

此时，索引的结构大概如下：

![img](https://pic3.zhimg.com/80/v2-043d10a60fcc2a4a5ee9d696a7537cae_1440w.jpg)

哈希索引中存储的是：哈希值+数据行指针

当你执行 SELECT lname FROM testhash WHERE fname='Peter'; MySQL会计算’Peter’的hash值，然后通过它来查询索引的行指针。因为f('Peter') = 8784，MySQL会在索引中查找8784，得到指向记录3的指针。

**Hash索引有以下一些限制：**

- 由于索引仅包含hash code和记录指针，所以，MySQL不能通过使用索引避免读取记录，即每次使用哈希索引查询到记录指针后都要回读元祖查取数据。
- 不能使用hash索引排序。
- Hash索引不支持键的部分匹配，因为是通过整个索引值来计算hash值的。
- Hash索引只支持等值比较，例如使用=，IN( )和<=>。对于WHERE price>100并不能加速查询。
- 访问Hash索引的速度非常快，除非有很多哈希冲突（不同的索引列值却有相同的哈希值）。当出现哈希冲突的时候，存储引擎必须遍历链表中所有的行指针，逐行进行比较，直到找到所有符合条件的行。
- 如果哈希冲突很多的话，一些索引维护操作的代价也会很高。当从表中删除一行时，存储引擎要遍历对应哈希值的链表中的每一行，找到并删除对应行的引用，冲突越多，代价越大。

InnoDB引擎有一个特殊的功能叫做“自适应哈希索引”，由Mysql自动管理，不需要DBA人为干预。默认情况下为开启，我们可以通过参数innodb_adaptive_hash_index来禁用此特性。

当InnoDB注意到某些索引值被使用得非常频繁时，它会在内存中基于缓冲池中的B+ Tree索引上再创建一个哈希索引，这样就上B-Tree索引也具有哈希索引的一些优点，比如快速的哈希查找。

- 只能用于等值比较，例如=， <=>，in ；
- 无法用于排序

InnoDB官方文档显示，启用自适应哈希索引后，读和写性能可以提高2倍，对于辅助索引的连接操作，性能可以提高5倍

**补充：索引存储在文件系统中**
索引是占据物理空间的，在不同的存储引擎中，索引存在的文件也不同。存储引擎是基于表的，以下分别使用MyISAM和InnoDB存储引擎建立两张表。
![存储引擎是基于表的，以下建立两张别使用MyISAM和InnoDB引擎的表，看看其在文件系统中对应的文件存储格式。](https://img-blog.csdnimg.cn/20210130233352294.png)

#### 存储引擎为MyISAM：

> *.frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
> *.MYD：MyISAM DATA，用于存储MyISAM表的数据
> *.MYI：MyISAM INDEX，用于存储MyISAM表的索引相关信息

#### 存储引擎为InnoDB：

> *.frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
> *.ibd：InnoDB DATA，表数据和索引的文件。该表的索引(B+树)的每个非叶子节点存储索引，叶子节点存储索引和索引对应的数据

## 索引分类

MySQL 的索引有两种分类方式：逻辑分类和物理分类

### 逻辑分类

有多种逻辑划分的方式，比如按功能划分，按组成索引的列数划分等

#### 按功能划分

**主键索引：**一张表只能有一个主键索引，不允许重复、不允许为 NULL；

```
 ALTER TABLE TableName ADD PRIMARY KEY(column_list); 
```

**唯一索引：**数据列不允许重复，允许为 NULL 值，一张表可有多个唯一索引，索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。

```MySQL
CREATE UNIQUE INDEX IndexName ON `TableName`(`字段名`(length));
```

或者

```MySQL
ALTER TABLE TableName ADD UNIQUE (column_list); 
```

**普通索引**：一张表可以创建多个普通索引，一个普通索引可以包含多个字段，允许数据重复，允许 NULL 值插入；

```MySQL
CREATE INDEX IndexName ON `TableName`(`字段名`(length));
```

或者

```MySQL
ALTER TABLE TableName ADD INDEX IndexName(`字段名`(length));
```

**全文索引**：它查找的是文本中的关键词，主要用于全文检索。

### 按列数划分

> - 单例索引：一个索引只包含一个列，一个表可以有多个单例索引。
> - 组合索引：一个组合索引包含两个或两个以上的列。查询的时候遵循 mysql 组合索引的 “最左前缀”原则，即使用 where 时条件要按照建立索引的时候字段的排列方式放置索引才会生效。

### 物理分类

分为聚簇索引和非聚簇索引（有时也称辅助索引或二级索引）

> 聚簇是为了提高某个属性(或属性组)的查询速度，把这个或这些属性(称为聚簇码)上具有相同值的元组集中存放在连续的物理块。
>
> 聚簇索引（clustered index）不是单独的一种索引类型，而是一种数据存储方式。这种存储方式是依靠B+树来实现的，根据表的主键构造一棵B+树且B+树叶子节点存放的都是表的行记录数据时，方可称该主键索引为聚簇索引。聚簇索引也可理解为将数据存储与索引放到了一块，找到索引也就找到了数据。
>
> 非聚簇索引：数据和索引是分开的，B+树叶子节点存放的不是数据表的行记录。
>

​		虽然InnoDB和MyISAM存储引擎都默认使用B+树结构存储索引，但是只有InnoDB的主键索引才是聚簇索引，InnoDB中的辅助索引以及MyISAM使用的都是非聚簇索引。每张表最多只能拥有一个聚簇索引。

**优点**：

> 数据访问更快，因为聚簇索引将索引和数据保存在同一个B+树中，因此从聚簇索引中获取数据比非聚簇索引更快
> 聚簇索引对于主键的排序查找和范围查找速度非常快

**缺点**：

> 插入速度严重依赖于插入顺序，按照主键的顺序插入是最快的方式，否则将会出现页分裂，严重影响性能。因此，对于InnoDB表，我们一般都会定义一个自增的ID列为主键（主键列不要选没有意义的自增列，选经常查询的条件列才好，不然无法体现其主键索引性能）
> .更新主键的代价很高，因为将会导致被更新的行移动。因此，对于InnoDB表，我们一般定义主键为不可更新。
> 二级索引访问需要两次索引查找，第一次找到主键值，第二次根据主键值找到行数据。

**Mysql中key 、primary key 、unique key 与index区别**
**key 与 index 含义**

> key具有两层含义：1.约束（约束和规范数据库的结构完整性）2.索引
>
> index：索引

**key 种类**

> key：等价普通索引 key 键名 (列)
>
> primary key：
>
> 约束作用（constraint），主键约束（unique，not null，一表一主键，唯一标识记录），规范存储主键和强调唯一性
> 为这个key建立主键索引
> unique key：
>
> 约束作用（constraint），unique约束（保证列或列集合提供了唯一性）
> 为这个key建立一个唯一索引；
> foreign key：
>
> 约束作用（constraint），外键约束，规范数据的引用完整性
> 为这个key建立一个普通索引；
>
> **如果一个Key有多个约束，将显示约束优先级最高的， PRI>UNI>MUL**

## **InnoDB和MyISAM索引实现**

## InnoDB索引实现

> InnoDB使用B+TREE存储数据，除了主键索引为聚簇索引，其它索引均为非聚簇索引。
>
> **一个表中只能存在一个聚簇索引（主键索引），但可以存在多个非聚簇索引。**
>
> InnoDB表的索引和数据是存储在一起的，`.idb`表数据和索引的文件
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210330051039780.png)

### 聚簇索引（主键索引）

​		B+树 叶子节点包含数据表中行记录就是聚簇索引（索引和数据是存放在一块的）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201102538557.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

​		可以看到叶子节点包含了完整的数据记录，这就是聚簇索引。因为InnoDB的数据文件（.idb）按主键聚集，所以InnoDB必须有主键（MyISAM可以没有），如果没有显示指定主键，则选取首个为唯一且非空的列作为主键索引，如果还没具备，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整型。

**主键索引结构分析：**

- B+树单个叶子节点内的行数据按主键顺序排列，物理空间是连续的（聚簇索引的数据的物理存放顺序与索引顺序是一致的）；
- 叶子节点之间是通过指针连接，相邻叶子节点的数据在逻辑上也是连续的（根据主键值排序），实际存储时的数据页（叶子节点）可能相距甚远。

### 非聚簇索引（辅助索引或二级索引）

​		在聚簇索引之外创建的索引（不是根据主键创建的）称之为辅助索引，辅助索引访问数据总是需要二次查找。辅助索引叶子节点存储的不再是行数据记录，而是主键值。首先通过辅助索引找到主键值，然后到主键索引树中通过主键值找到数据行

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201102648935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

**InnoDB索引优化**

> - InnoDB中主键不宜定义太大，因为辅助索引也会包含主键列，如果主键定义的比较大，其他索引也将很大。如果想在表上定义 、很多索引，则争取尽量把主键定义得小一些。InnoDB 不会压缩索引。
>
>
> - InnoDB中尽量不使用非单调字段作主键（不使用多列），因为InnoDB数据文件本身是一颗B+Tree，非单调的主键会造成在插入新记录时数据文件为了维持B+Tree的特性而频繁的分裂调整，十分低效，而使用自增字段作为主键则是一个很好的选择。
>   

## MyISAM索引实现

> MyISAM也使用B+Tree作为索引结构，但具体实现方式却与InnoDB截然不同。MyISAM使用的都是非聚簇索引。
>
> MyISAM表的索引和数据是分开存储的，`.MYD`表数据文件 `.MYI`表索引文件
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201165250992.png)

### MyISAM主键索引

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201170401133.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

> 可以看到叶子节点的存放的是数据记录的地址。也就是说索引和行数据记录是没有保存在一起的，所以MyISAM的主键索引是非聚簇索引。

###  MyISAM辅助索引

> 在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。 MyISAM辅助索引也是非聚簇索引。

##  聚簇索引和非聚簇索引的区别 

> - 聚簇索引的叶子节点存放的是数据行（主键值也是行内数据），支持覆盖索引；而二级索引的叶子节点存放的是主键值或指向数据行的指针。
> - 由于叶子节点(数据页)只能按照一棵B+树排序，故一张表只能有一个聚簇索引。辅助索引的存在不影响聚簇索引中数据的组织，所以一张表可以有多个辅助索引。

## 前缀索引 

> 有时候需要索引很长的字符列，这会让索引变得大且慢。通常可以以某列开始的部分字符作为索引，这样可以大大节约索引空间，从而提高索引效率。但这样也会降低索引的选择性。索引的选择性是指不重复的索引值和数据表的记录总数的比值，索引的选择性越高则查询效率越高。
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206144840714.png#pic_center)
>
> 不使用索引查询某条记录
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206162906197.png)
>
> 使用索引查询某条记录（前缀字符并非越多越好，需要在索引的选择性和索引IO读取量中做出衡量。）
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206163004906.png)
>
> ```MySQL
> alter table x_test add index(x_name(4));
> ```
>
> **需要注意的一点是，前缀索引不能使用覆盖索引**

## 全文索引 FULLTEXT

> 通过数值比较、范围过滤等就可以完成绝大多数我们需要的查询，但是，如果希望通过关键字的匹配来进行查询过滤，那么就需要基于相似度的查询，而不是原来的精确数值比较。全文索引，就是为这种场景设计的，通过建立倒排索引,可以极大的提升检索效率,解决判断字段是否包含的问题

### FULLTEXT VS LIKE+%

​		使用LIKE+%确实可以实现模糊匹配，适用于文本比较少的时候。对于大量的文本检索，LIKE+%与全文索引的检索速度相比就不是一个数量级的。

​		例如: 有title字段，需要查询所有包含 "政府"的记录.，使用 like "%政府%“方式查询，查询速度慢（全表查询）。且当查询包含"政府” OR "中国"的字段时，使用like就难以简单满足，而全文索引就可以实现这个功能。

### FULLTEXT的支持情况

> MySQL 5.6 以前的版本，只有 MyISAM 存储引擎支持全文索引；
> MySQL 5.6 及以后的版本，MyISAM 和 InnoDB 存储引擎均支持全文索引;
> 只有字段的数据类型为 char、varchar、text 及其系列才可以建全文索引
> InnoDB内部并不支持中文、日文等，因为这些语言没有分隔符。可以使用插件辅助实现中文、日文等的全文索引。

### 创建全文索引

```MySQL
//建表的时候
FULLTEXT KEY keyname(colume1,colume2)  // 创建联合全文索引列

//在已存在的表上创建
create fulltext index keyname on xxtable(colume1,colume2);

alter table xxtable add fulltext index keyname (colume1,colume2);

```

### 使用全文索引

​		全文索引有独特的语法格式，需要配合match 和 against 关键字使用

- **match()函数中指定的列必须是设置为全文索引的列**
  **against()函数标识需要模糊查找的关键字**

```MySQL
 create table fulltext_test(
     id int auto_increment primary key,
     words varchar(2000) not null,a
     artical text not null,
     fulltext index words_artical(words,artical)
)engine=innodb default charset=utf8;

insert into fulltext_test values(null,'a','a');
insert into fulltext_test values(null,'aa','aa');
insert into fulltext_test values(null,'aaa','aaa');
insert into fulltext_test values(null,'aaaa','aaaa');
```

好，我们使用全文搜索查看一下

```MySQL
select * from fulltext_test where match(words,artical) against('a');
select * from fulltext_test where match(words,artical) against('aa');
select * from fulltext_test where match(words,artical) against('aaa');
select * from fulltext_test where match(words,artical) against('aaaa');
```

发现只有aaa和aaaa才能查到记录，为什么会这样呢？

### 全文索引关键词长度阈值

​		这其实跟全文搜索的关键词的长度阈值有关，可以通过`show variables like '%ft%';`查看。可见InnoDB的全文索引的关键词 最小索引长度 为3。上文使用的是InnoDB引擎建表，同时也解释为什么只有3a以上才有搜索结果

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210207204117449.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

> 设置关键词长度阈值，可以有效的避免过短的关键词，得到的结果过多也没有意义。
>
> 也可以手动配置关键词长度阈值，修改MySQL配置文件，在[mysqld]的下面追加以下内容，设置关键词最小长度为5

然后重启MySQL服务器，还要修复索引，不然参数不会生效

```MySQL
repair table 表名 quick;
```

为什么上文搜索关键字为aaa的时候，有且仅有一条aaa的记录，为什么没有aaaa的记录呢？

###  全文索引模式

自然语言的全文索引 IN NATURAL LANGUAGE MODE

默认情况下，或者使用 IN NATURAL LANGUAGE MODE 修饰符时，match() 函数对文本集合执行自然语言搜索，上面的例子都是自然语言的全文索引。

自然语言搜索引擎将计算每一个文档对象和查询的相关度。这里，相关度是基于匹配的关键词的个数，以及关键词在文档中出现的次数。在整个索引中出现次数越少的词语，匹配时的相关度就越高。

MySQL在全文查询中会对每个合适的词都会先计算它们的权重，如果一个词出现在多个记录中，那它只有较低的权重；相反，如果词是较少出现在这个集的文档中，它将得到一个较高的权重。

MySQL默认的阀值是50%。如果一个词语的在超过 50% 的记录中都出现了，那么自然语言的搜索将不会搜索这类词语。

上文关键词长度阈值是3，所以相当于只有两条记录：aaa 和 aaaa
aaa 权重 2/2=100% 自然语言的搜索将不会搜索这类词语aaa了 而是进行精确查找 aaaa不会出现在aaa的结果集中。

这个机制也比较好理解，比如一个数据表存储的是一篇篇的文章，文章中的常见词、语气词等等，出现的肯定比较多，搜索这些词语就没什么意义了，需要搜索的是那些文章中有特殊意义的词，这样才能把文章区分开。

```MySQL
select * from fulltext_test where match(words,artical) against('aaa');
等价
select * from fulltext_test where match(words,artical) against('aaa'  IN NATURAL LANGUAGE MODE);
```

布尔全文索引 IN BOOLEAN MODE

在布尔搜索中，我们可以在查询中自定义某个被搜索的词语的相关性，这个模式和lucene中的BooleanQuery很像,可以通过一些操作符,来指定搜索词在结果中的包含情况。

建立如下表：

```MySQL
CREATE TABLE articles (
    id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
    title VARCHAR(200),
    body TEXT,
    FULLTEXT (title,body)
) ENGINE=InnoDB
```


+ （AND）全文索引列必须包含该词且全文索引列（之一）有且仅有该词

- （NOT）表示必须不包含,默认为误操作符。如果只有一个关键词且使用了- ，会将这个当成错误操作，相当于没有查询关键词；如果有多个关键词，关键词集合中不全是负能符（~ -），那么-则强调不能出现。

-- 查找title,body列中有且仅有apple（是apple不是applexx 也不是 xxapple）但是不含有banana的记录
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+apple -banana' IN BOOLEAN MODE);
1
2
> 提高该词的相关性，查询的结果靠前
> < 降低该词的相关性，查询的结果靠后
> -- 返回同时包含apple（是apple不是applexx 也不是 xxapple）和banana或者同时包含apple和orange的记录。但是同时包含apple和banana的记录的权重高于同时包含apple和orange的记录。
> SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+apple +(>banana <orange)' IN BOOLEAN MODE);   
> 1
> 2
> ~ 异或，如果包含则降低关键词整体的相关性
> -- 返回的记录必须包含apple（且不能是applexx 或 xxapple），但是如果同时也包含banana会降低权重（只出现apple的记录会出现在结果集前面）。但是它没有 +apple -banana 严格，因为后者如果包含banana压根就不返回。
> SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+apple ~banana' IN BOOLEAN MODE);
> 1
> 2
* 通配符，表示关键词后面可以跟任意字符
-- 返回的记录可以为applexx
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('apple*' IN BOOLEAN MODE);
1
2
空格 表示OR
-- 查找title,body列中包含apple（是apple不是applexx 也不是 xxapple）或者banana的记录，至少包含一个
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('apple banana' IN BOOLEAN MODE);  
1
2
"" 双引号，效果类似like '%some words%'
-- 模糊匹配 “apple banana goog”会被匹配到，而“apple good banana”就不会被匹配
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('"apple banana"' IN BOOLEAN MODE);  
1
2
InnoDB中FULLTEXT中文支持
InnoDB内部并不支持中文、日文等，因为这些语言没有分隔符。可以使用插件辅助实现中文、日文等的全文索引。

MySQL内置ngram插件便可解决该问题。

```MySQL
 FULLTEXT (title, body) WITH PARSER ngram
ALTER TABLE articles ADD FULLTEXT INDEX ft_index (title,body) WITH PARSER ngram;
CREATE FULLTEXT INDEX ft_index ON articles (title,body) WITH PARSER ngram;
```

## **空间(R-Tree)索引**

MyISAM支持空间索引，主要用于地理空间数据类型，例如GEOMETRY

## **索引排序**

也可以利用B-Tree索引进行索引排序（对查询结果进行ORDER BY），必须保证ORDER BY按索引的最左边前缀(leftmost prefix of the index)来进行。

MySQL中，有两种方式生成有序结果集：

- 使用filesort
- 按索引顺序扫描

如果explain出来的type列的值为“index”，则说明MYSQL使用了索引扫描来做排序。

**按索引顺序扫描：**

可以利用同一索引同时进行查找和排序操作：

- 当索引的顺序与ORDER BY中的列顺序相同，且所有的列是同一方向（全部升序或者全部降序）时，可以使用索引来排序。
- ORDER BY子句和查询型子句的限制是一样的：需要满足索引的最左前缀的要求，有一种情况下ORDER BY子句可以不满足索引的最左前缀要求，那就是前导列为常量时：WHERE子句或者JOIN子句中对前导列指定了常量。

![img](https://pic1.zhimg.com/80/v2-19d7202ee71287a606f644837d0c19cc_1440w.jpg)

- 如果查询是连接多个表，仅当ORDER BY中的所有列都是第一个表的列时才会使用索引。其它情况都会使用filesort文件排序。

**使用filesort：**

当MySQL不能使用索引进行排序时，就会利用自己的排序算法(快速排序算法)在内存(sort buffer)中对数据进行排序；

如果内存装载不下，它会将磁盘上的数据进行分块，再对各个数据块进行排序，然后将各个块合并成有序的结果集（实际上就是外排序，使用临时表）。

对于**filesort**，MySQL有两种排序算法：

- **两次扫描算法(Two passes)**

先将需要排序的字段和可以直接定位到相关行数据的指针信息取出，然后在设定的内存（通过参数sort_buffer_size设定）中进行排序，完成排序之后再次通过行指针信息取出所需的Columns。

该算法是MySQL4.1之前采用的算法，它需要两次访问数据，尤其是第二次读取操作会导致大量的随机I/O操作。另一方面，内存开销较小。

- **一次扫描算法(single pass)**

该算法一次性将所需的Columns全部取出，在内存中排序后直接将结果输出。

从MySQL4.1版本开始使用该算法。它减少了I/O的次数，效率较高，但是内存开销也较大。如果我们将并不需要的Columns也取出来，就会极大地浪费排序过程所需要的内存。

在 MySQL 4.1 之后的版本中，可以通过设置 max_length_for_sort_data 参数来控制 MySQL 选择第一种排序算法还是第二种：当取出的所有大字段总大小大于 max_length_for_sort_data 的设置时，MySQL 就会选择使用第一种排序算法，反之，则会选择第二种。

当对连接操作进行排序时，如果ORDER BY仅仅引用第一个表的列，MySQL对该表进行filesort操作，然后进行连接处理，此时，EXPLAIN输出“Using filesort”；否则，MySQL必须将查询的结果集生成一个临时表，在连接完成之后进行filesort操作，此时，EXPLAIN输出“Using temporary;Using filesort”。

为了尽可能地提高排序性能，我们自然更希望使用第二种排序算法，所以在 Query 中仅仅取出需要的 Columns 是非常有必要的。

## **索引使用**

## **MySQL建立索引类型**

- 单列索引，即一个索引只包含单个列，一个表可以有多个单列索引，但这不是组合索引。
- 组合索引，即一个索包含多个列。

索引是在存储引擎中实现的，而不是在服务器层中实现的。所以，每种存储引擎的索引都不一定完全相同，并不是所有的存储引擎都支持所有的索引类型。

## **普通索引**

这是最基本的索引，它没有任何限制。普通索引（由关键字KEY或INDEX定义的索引）的唯一任务是加快对数据的访问速度。因此，应该只为那些最经常出现在查询条件(WHERE column = …)或排序条件(ORDER BY column)中的数据列创建索引。

它有以下几种创建方式：

- 创建索引

CREATE INDEX indexName ON mytable(username(length));

如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length，下同。

- 修改表结构

ALTER mytable ADD INDEX [indexName] ON (username(length))

- 创建表的时候直接指定

CREATE TABLE mytable( ID INT NOT NULL, username VARCHAR(16) NOT NULL, INDEX [indexName] (username(length)) );

- 删除索引的语法：

DROP INDEX [indexName] ON mytable;

## **唯一索引**

它与前面的普通索引类似，不同的就是：普通索引允许被索引的数据列包含重复的值。而唯一索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。

它有以下几种创建方式：

- 创建索引

CREATE UNIQUE INDEX indexName ON mytable(username(length))

- 修改表结构

ALTER mytable ADD UNIQUE [indexName] ON (username(length))

- 创建表的时候直接指定

CREATE TABLE mytable( ID INT NOT NULL, username VARCHAR(16) NOT NULL, UNIQUE [indexName] (username(length)) );

## **主键索引**

它是一种特殊的唯一索引，不允许有空值。一个表只能有一个主键。

一般是在建表的时候同时创建主键索引：

CREATE TABLE mytable( ID INT NOT NULL, username VARCHAR(16) NOT NULL, PRIMARY KEY(ID) ); 当然也可以用 ALTER 命令。

与之类似的，外键索引

如果为某个外键字段定义了一个外键约束条件，MySQL就会定义一个内部索引来帮助自己以最有效率的方式去管理和使用外键约束条件。

## **组合索引**

为了形象地对比单列索引和组合索引，为表添加多个字段：

CREATE TABLE mytable( ID INT NOT NULL, username VARCHAR(16) NOT NULL, city VARCHAR(50) NOT NULL, age INT NOT NULL );

为了进一步榨取MySQL的效率，就要考虑建立组合索引。就是将 name, city, age建到一个索引里：

ALTER TABLE mytable ADD INDEX name_city_age (name(10),city,age);

建表时，usernname长度为 16，这里用 10。这是因为一般情况下名字的长度不会超过10，这样会加速索引查询速度，还会减少索引文件的大小，提高INSERT的更新速度。

建立这样的组合索引，其实是相当于分别建立了下面三组组合索引：

usernname,city,age

usernname,city

usernname

为什么没有 city，age这样的组合索引呢？这是因为MySQL组合索引“最左前缀”的结果。简单的理解就是只从最左面的开始组合。并不是只要包含这三列的查询都会用到该组合索引。下面的几个SQL就会用到这个组合索引：

SELECT * FROM mytable WHREE username="admin" AND city="郑州"

SELECT * FROM mytable WHREE username="admin"

而下面几个则不会用到：

SELECT * FROM mytable WHREE age=20 AND city="郑州"

SELECT * FROM mytable WHREE city="郑州"

如果分别在 usernname，city，age上建立单列索引，让该表有3个单列索引，查询时和上述的组合索引效率也会大不一样，远远低于我们的组合索引。因为虽然此时有了三个索引，但MySQL只能用到其中的那个它认为似乎是最有效率的单列索引。

## **建立索引的时机**

一般来说，在WHERE和JOIN中出现的列需要建立索引，但也不完全如此，因为MySQL的B-Tree只对<，<=，=，>，>=，BETWEEN，IN，以及不以通配符开始的LIKE才会使用索引。

例如：

SELECT t.Name FROM mytable t LEFT JOIN mytable m ON t.Name=m.username WHERE m.age=20 AND m.city='郑州'

此时就需要对city和age建立索引，由于mytable表的userame也出现在了JOIN子句中，也有对它建立索引的必要。