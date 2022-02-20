**binlog（二进制日志）**

**定义：**

逻辑格式的日志，可以认为就是执行过事物sql语句，包含了执行sql语句（增删改）的反向信息

binlog作为还原的功能，是数据库层面的（当然也可以精确到事务层面的），可以作为恢复数据使用，主从复制搭建

binlog是追加写，是指一份写到一定大小的时候会更换下一个文件，不会覆盖

**生成时间：**

事物提交的时候，一次性将事物中的sql语句按照一定的格式存入binlog中

**释放时间：**

binlog的默认是保持时间由参数expire_logs_days配置，也就是说对于非活动的日志文件，在生成时间超过expire_logs_days配置的天数之后，会被自动删除

**物理文件：**

配置文件的路径为log_bin_basename，binlog日志文件按照指定大小，当日志文件达到指定的最大的大小之后，进行滚动更新，生成新的日志文件，对于每个binlog日志文件，通过一个统一的index文件来组织

**redolog（重做日志）**

**定义:**

物理格式的日志，记录的是物理数据页面的修改的信息

redo log是保证事务的持久性的，是事务层面的，作为异常宕机或者介质故障后的数据恢复使用

redo log是循环写，日志空间大小固定（mysql默认16kb，日志每次写入数据页大小为扇区page大小512字节）

每个Page的修改对应一个Mtr（Mini Transaction）底层物理事物

**Write-Ahead Log**

先在内存中提交事务，然后写日志（所谓的Write-ahead Log），然后后台任务把内存中的数据异步刷到磁盘。日志是顺序地在尾部Append，从而也就避免了一个事务发生多次磁盘随机 I/O 的问题

在InnoDB中，不光事务修改的数据库表数据是异步刷盘的，连Redo Log的写入本身也是异步的

**刷盘策略**

InnoDB有个关键的参数innodb_flush_log_at_trx_commit控制 Redo Log的刷盘策略，该参数有三个取值

- 0：每秒刷一次磁盘，把Redo Log Buffer中的数据刷到Redo Log（默认为0）。
- 1：每提交一个事务，就刷一次磁盘（这个最安全）。
- 2：不刷盘。然后根据参数innodb_flush_log_at_timeout设置的值决 定刷盘频率。

该参数设置为0或者2都可能丢失数据。设置为1最安全， 但性能最差。InnoDB设置此参数，也是为了让应用在数据安全性和性 能之间做一个权衡

**LSN（Log Sequence Number）**

逻辑上日志按照时间顺序从小到大的编号。在InnoDB中， LSN是一个64位的整数，取的是从数据库安装启动开始，到当前所写入 的总的日志字节数。实际上LSN没有从0开始，而是从8192开始，这个 是InnoDB源代码里面的一个常量LOG_START_LSN。因为事务有大有 小，每个事务产生的日志数据量是不一样的，所以日志是变长记录，因 此LSN是单调递增的，但肯定不是呈单调连续递增

**Physiological** **Logging**

- 记法1：类似Binlog的statement格式，记原始的SQL语句， insert/delete/update。
- 记法2：类似Binlog的RAW格式，记录每张表的每条记录的修 改前的值、修改后的值，类似（表，行，修改前的值，修改后的值）。
- 记法3：记录修改的每个Page的字节数据。由于每个Page有 16KB，记录这16KB里哪些部分被修改了。一个Page如果被修改了多个 地方，就会有多条物理日志，如下所示： （PageID,offset1,len1，改之前的值，改之后的值）

（Page ID,offset2,len2，改之前的值，改之后的值）

前两种记法都是逻辑记法；第三种是物理记法Redo Log采用了逻辑和物理的综合体，就是先以Page为单位记录日 志，每个Page里面再采取逻辑记法（记录Page里面的哪一行被修改 了）。这种记法有个专业术语，叫Physiological Logging。

**IO写入的原子性（Double Write）**

-  让硬件支持16KB写入的原子性。要么写入0个字节，要么 16KB全部成功。
-  Double write。把16KB写入到一个临时的磁盘位置，写入成功后再拷贝到目标磁盘位置。 这样，即使目标磁盘位置的16KB因为宕机被损坏了，还可以用备份去恢复

**Redo Log Block结构**

一个Redo Log Block的详细结构，头部有12字节，尾部 Check sum有4个字节，所以实际一个Block能存的日志数据只有496字节

头部4个字段的含义分别如下：

- **Block No**：每个Block的唯一编号，可以由LSN换算得到。
- **Date Len**：该Block中实际日志数据的大小，可能496字节没有存 满。
- **First Rec Group**：该 Block 中第一条日志的起始位置，可能因为上 一条日志很大，上一个Block没有存下，日志的部分数据到了当前的 Block。如果First Rec Group = Data Len，则说明上一条日志太大，大到 横跨了上一个Block、当前Block、下一个Block，当前Block中没有新日 志。
- **Checkpoint No**：当前Block进行Check point时对应的LSN

**事务Rollback与崩溃恢复（ARIES算法）**

**1．未提交事务的日志也在Redo Log中**

不管事务有没有提交，其日志 都会被记录到Redo Log上。当崩溃后再恢复的时候，会把Redo Log全部 重放一遍，提交的事务和未提交的事务，都被重放了，从而让数据 库“原封不动”地回到宕机之前的状态，这叫Repeating History

**2.Rollback转化为Commit**

这种逆向操作的SQL语句对应到Redo Log里面，叫作Compensation Log Record（CLR），会和正常操作的 SQL的Log区分开

**3.ARIES恢复算法**

- **阶段1：分析阶段**

 Fuzzy Checkpoint每隔一段时间对 内存中的数据拍一个“快照”

 在内存中，维护了两个关键的表：**活跃事务表**和**脏页表**

 **活跃事务表**是当前所有未提交事务的集合，每个事务维护了一个关键变量**lastLSN**，是    

 该事务产生的日志中最后一条日志的LSN

 **脏页表**是当前所有未刷到磁盘上的Page的集合（包括了已提交的事 务和未提交的事务）

每一次 Fuzzy Checkpoint，就把两个表的数据生成一个快照，形成一条 Checkpoint日志，记入Redo Log，recoveryLSN是导致该Page为脏页的最早的LSN，

- **阶段2：进行Redo**

每个Page有一个关键字段—— pageLSN日志的 LSN <=pageLSN，则不修改 日志对应的Page，略过此条日志，保证redo log幂等

- **阶段3：进行Undo**

**Undo Log（回滚日志）**

保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读

事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性

当事务提交之后，undo log并不能立马被删除，

而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间

innodb_undo_logs = 128 –回滚段为128KB

**Undo + Redo事务的特点**

 A. 为了保证持久性，必须在事务提交前将Redo Log持久化。

 B. 数据不需要在事务提交前写入磁盘，而是缓存在内存中。

 C. Redo Log 保证事务的持久性。

 D. Undo Log 保证事务的原子性。

 E. 有一个隐含的特点，数据必须要晚于redo log写入持久存储