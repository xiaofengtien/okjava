**什么是MVCC**

MVCC (Multiversion Concurrency Control) 中文全程叫多版本并发控制，目的在于提高数据库高并发场景下的吞吐性能

**为什么要使用MVCC**

InnoDB 相比 MyISAM 有两大特点，一是支持事务而是支持行级锁，

但并发事务处理也会带来一些问题，主要包括以下几种情况：

1. 更新丢失（Lost Update）：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题 —— 最后的更新覆盖了其他事务所做的更新。如何避免这个问题呢，最好在一个事务对数据进行更改但还未提交时，其他事务不能访问修改同一个数据。
2. 脏读（Dirty Reads）：一个事务正在对一条记录做修改，在这个事务并提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些尚未提交的脏数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象被形象地叫做 “脏读”。
3. 不可重复读（Non-Repeatable Reads）：一个事务在读取某些数据已经发生了改变、或某些记录已经被删除了！这种现象叫做“不可重复读”。
4. 幻读（Phantom Reads）：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为 “幻读”。

==实现隔离机制==的方法主要有两种:

1. 加读写锁
2. 一致性快照读，即 MVCC

**InnoDB 中的 MVCC**

1. MySQL 中 InnoDB 引擎支持 MVCC
2. 应对高并发事务, MVCC 比单纯的加行锁更有效, 开销更小
3. MVCC 在读已提交（Read Committed）和可重复读（Repeatable Read）隔离级别下起作用
4. MVCC 既可以基于乐观锁又可以基于悲观锁来实现

**InnoDB MVCC 实现原理**

InnoDB 中 MVCC 的实现方式为：每一行记录都有两个隐藏列：DATA_TRX_ID、DATA_ROLL_PTR（如果没有主键，则还会多一个隐藏的主键列、DB_ROW_ID

**DATA_TRX_ID**

记录最近更新这条行记录的事务 ID，大小为 6 个字节

**DATA_ROLL_PTR**

表示指向该行回滚段（rollback segment）的指针，大小为 7 个字节，InnoDB 便是通过这个指针找到之前版本的数据。该行记录上所有旧版本，在 undo 中都通过链表的形式组织。

**DB_ROW_ID**

行标识（隐藏单调自增 ID），大小为 6 字节，如果表没有主键，InnoDB 会自动生成一个隐藏主键，因此会出现这个列。另外，每条记录的头信息（record header）里都有一个专门的 bit（deleted_flag）来表示当前记录是否已经被删除。

**update操作：**

1. 对行记录加排他锁
2. 把该行原本的值拷贝到 undo log 中，DB_TRX_ID 和 DB_ROLL_PTR 都不动
3. 修改该行的值这时产生一个新版本，更新 DATA_TRX_ID 为修改记录的事务 ID，将 DATA_ROLL_PTR 指向刚刚拷贝到 undo log 链中的旧版本记录，这样就能通过 DB_ROLL_PTR 找到这条记录的历史版本。如果对同一行记录执行连续的 UPDATE，Undo Log 会组成一个链表，遍历这个链表可以看到这条记录的变迁
4. 记录 redo log，包括 undo log 中的修改

**INSERT 和 DELETE** 

其实相比 UPDATE 这二者很简单，INSERT 会产生一条新纪录，它的 DATA_TRX_ID 为当前插入记录的事务 ID；

DELETE 某条记录时可看成是一种特殊的 UPDATE，其实是软删，真正执行删除操作会在 commit 时，DATA_TRX_ID 则记录下删除该记录的事务 ID

**一致性读 —— ReadView**	

核心问题是版本链中哪些版本对当前事务可见

ReadView 中是当前活跃的事务 ID 列表（未提交事物）

**RR 下的 ReadView 生成**	

（begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果你想要马上启动一个事务，可以使用 start transaction with consistent snapshot 这个命令）

在 RR 隔离级别下，每个事务 touch first read 时（本质上就是执行第一个 SELECT 语句时，后续所有的 SELECT 都是复用这个 ReadView，其它 update, delete, insert 语句和一致性读 snapshot 的建立没有关系），会将当前系统中的所有的活跃事务拷贝到一个列表生成ReadView，事物id更新不会影响当前m_ids表的最大值和最小值

**RC 下的 ReadView 生成**

在 RC 隔离级别下，每个 SELECT 语句开始时，都会重新将当前系统中的所有的活跃事务拷贝到一个列表生成 ReadView。二者的区别就在于生成 ReadView 的时间点不同，一个是事务之后第一个 SELECT 语句开始、一个是事务中每条 SELECT 语句开始。

ReadView 中是当前活跃的事务 ID 列表，称之为 m_ids，其中最小值为 up_limit_id，最大值为 low_limit_id，事务 ID 是事务开启时 InnoDB 分配的，其大小决定了事务开启的先后顺序，因此我们可以通过 ID 的大小关系来决定版本记录的可见性，具体判断流程如下：

1. 如果被访问版本的 trx_id 小于 m_ids 中的最小值 up_limit_id，说明生成该版本的事务在 ReadView 生成前就已经提交了，所以该版本可以被当前事务访问。
2. 如果被访问版本的 trx_id 大于 m_ids 列表中的最大值 low_limit_id，说明生成该版本的事务在生成 ReadView 后才生成，所以该版本不可以被当前事务访问。需要根据 Undo Log 链找到前一个版本，然后根据该版本的 DB_TRX_ID 重新判断可见性。
3. 如果被访问版本的 trx_id 属性值在 m_ids 列表中最大值和最小值之间（包含），那就需要判断一下 trx_id 的值是不是在 m_ids 列表中。如果在，说明创建 ReadView 时生成该版本所属事务还是活跃的，因此该版本不可以被访问，需要查找 Undo Log 链得到上一个版本，然后根据该版本的 DB_TRX_ID 再从头计算一次可见性；如果不在，说明创建 ReadView 时生成该版本的事务已经被提交，该版本可以被访问。
4. 此时经过一系列判断我们已经得到了这条记录相对 ReadView 来说的可见结果。此时，如果这条记录的 delete_flag 为 true，说明这条记录已被删除，不返回。否则说明此记录可以安全返回给客户端。

RC、RR 两种隔离级别的事务在执行普通的读操作时，通过访问版本链的方法，使得事务间的读写操作得以并发执行，从而提升系统性能。RC、RR 这两个隔离级别的一个很大不同就是生成 ReadView 的时间点不同，RC 在每一次 SELECT 语句前都会生成一个 ReadView，事务期间会更新，因此在其他事务提交前后所得到的 m_ids 列表可能发生变化，使得先前不可见的版本后续又突然可见了。而 RR 只在事务的第一个 SELECT 语句时生成一个 ReadView，事务操作期间不更新