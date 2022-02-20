**1 QPS计算(每秒查询数)**

针对MyISAM引擎为主的DB

| 123456789101112131415 | MySQL> show GLOBAL status like 'questions';+---------------+------------+\| Variable_name \| Value  \|+---------------+------------+\| Questions  \| 2009191409 \|+---------------+------------+1 row in set (0.00 sec) mysql> show global status like 'uptime';+---------------+--------+\| Variable_name \| Value \|+---------------+--------+\| Uptime  \| 388402 \|+---------------+--------+1 row in set (0.00 sec) |
| --------------------- | ------------------------------------------------------------ |
|                       |                                                              |

QPS=questions/uptime=5172，mysql自启动以来的平均QPS，如果要计算某一时间段内的QPS，可在高峰期间获取间隔时间t2-t1，然后分别计算出t2和t1时刻的q值，QPS=(q2-q1)/(t2-t1)

针对InnnoDB引擎为主的DB

| 123456789101112131415161718192021222324252627282930313233 | mysql> show global status like 'com_update';+---------------+----------+\| Variable_name \| Value \|+---------------+----------+\| Com_update \| 87094306 \|+---------------+----------+1 row in set (0.00 sec) mysql> show global status like 'com_select';+---------------+------------+\| Variable_name \| Value  \|+---------------+------------+\| Com_select \| 1108143397 \|+---------------+------------+1 row in set (0.00 sec) mysql> show global status like 'com_delete';+---------------+--------+\| Variable_name \| Value \|+---------------+--------+\| Com_delete \| 379058 \|+---------------+--------+1 row in set (0.00 sec) mysql> show global status like 'uptime'; +---------------+--------+\| Variable_name \| Value \|+---------------+--------+\| Uptime  \| 388816 \|+---------------+--------+1 row in set (0.00 sec) |
| --------------------------------------------------------- | ------------------------------------------------------------ |
|                                                           |                                                              |

QPS=(com_update+com_insert+com_delete+com_select)/uptime=3076,某一时间段内的QPS查询方法同上。

并发数=QPS*RT(处理时间间隔)

**2 TPS计算(每秒事务数)**

| 1234567891011121314151617181920212223242526 | mysql> show global status like 'com_commit'; +---------------+---------+\| Variable_name \| Value \|+---------------+---------+\| Com_commit \| 7424815 \|+---------------+---------+1 row in set (0.00 sec) mysql> show global status like 'com_rollback';+---------------+---------+\| Variable_name \| Value \|+---------------+---------+\| Com_rollback \| 1073179 \|+---------------+---------+1 row in set (0.00 sec) mysql> show global status like 'uptime';+---------------+--------+\| Variable_name \| Value \|+---------------+--------+\| Uptime  \| 389467 \|+---------------+--------+1 row in set (0.00 sec) TPS=(com_commit+com_rollback)/uptime=22 |
| ------------------------------------------- | ------------------------------------------------------------ |
|                                             |                                                              |

**3 线程连接数和命中率**

| 123456789101112131415161718192021222324252627282930 | mysql> show global status like 'threads_%';+-------------------+-------+\| Variable_name  \| Value \|+-------------------+-------+\| Threads_cached \| 480 \|  //代表当前此时此刻线程缓存中有多少空闲线程\| Threads_connected \| 153 \| //代表当前已建立连接的数量，因为一个连接就需要一个线程，所以也可以看成当前被使用的线程数\| Threads_created \| 20344 \| //代表从最近一次服务启动，已创建线程的数量\| Threads_running \| 2  \|  //代表当前激活的（非睡眠状态）线程数+-------------------+-------+4 rows in set (0.00 sec) mysql> show global status like 'Connections';+---------------+-----------+\| Variable_name \| Value  \|+---------------+-----------+\| Connections \| 381487397 \|+---------------+-----------+1 row in set (0.00 sec) 线程缓存命中率=1-Threads_created/Connections = 99.994% 我们设置的线程缓存个数 mysql> show variables like '%thread_cache_size%';+-------------------+-------+\| Variable_name  \| Value \|+-------------------+-------+\| thread_cache_size \| 500 \|+-------------------+-------+1 row in set (0.00 sec) |
| --------------------------------------------------- | ------------------------------------------------------------ |
|                                                     |                                                              |

根据Threads_connected可预估thread_cache_size值应该设置多大，一般来说250是一个不错的上限值，如果内存足够大，也可以设置成thread_cache_size值和threaads_connected值相同；

或者通过观察threads_created值，如果该值很大或一直在增长，可以适当增加thread_cache_size的值；在休眠状态下每个线程大概占用256KB左右的内存，所以当内存足够时，设置太小也不会节约太多内存，除非该值已经超过几千。

**4 表缓存**

| 1234567 | mysql> show global status like 'open_tables%';+---------------+-------+\| Variable_name \| Value \|+---------------+-------+\| Open_tables \| 2228 \|+---------------+-------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

我们设置的打开表的缓存和表定义缓存

| 123456789101112131415 | mysql> show variables like 'table_open_cache';+------------------+-------+\| Variable_name \| Value \|+------------------+-------+\| table_open_cache \| 16384 \|+------------------+-------+1 row in set (0.00 sec) mysql> show variables like 'table_defi%';+------------------------+-------+\| Variable_name   \| Value \|+------------------------+-------+\| table_definition_cache \| 2000 \|+------------------------+-------+1 row in set (0.00 sec) |
| --------------------- | ------------------------------------------------------------ |
|                       |                                                              |

**针对MyISAM：**

mysql每打开一个表，都会读入一些数据到table_open_cache 缓存 中，当mysql在这个缓存中找不到相应的信息时，才会去磁盘上直接读取，所以该值要设置得足够大以避免需要重新打开和重新解析表的定义，一般设置为max_connections的10倍，但最好保持在10000以内。

还有种依据就是根据状态open_tables的值进行设置，如果发现open_tables的值每秒变化很大，那么可能需要增大table_open_cache的值。

table_definition_cache 通常简单设置为服务器中存在的表的数量，除非有上万张表。

**针对InnoDB：**

与MyISAM不同，InnoDB的open table和open file并无直接联系，即打开frm表时其相应的ibd文件可能处于关闭状态；

故InnoDB只会用到table_definiton_cache，不会使用table_open_cache；

其frm文件保存于table_definition_cache中，而idb则由innodb_open_files决定(前提是开启了innodb_file_per_table)，最好将innodb_open_files设置得足够大，使得服务器可以保持所有的.ibd文件同时打开。

**5 最大连接数**

| 1234567 | mysql> show global status like 'Max_used_connections';+----------------------+-------+\| Variable_name  \| Value \|+----------------------+-------+\| Max_used_connections \| 1785 \|+----------------------+-------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

我们设置的max_connections大小

| 1234567 | mysql> show variables like 'max_connections%';+-----------------+-------+\| Variable_name \| Value \|+-----------------+-------+\| max_connections \| 4000 \|+-----------------+-------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

通常max_connections的大小应该设置为比Max_used_connections状态值大，Max_used_connections状态值反映服务器连接在某个时间段是否有尖峰，如果该值大于max_connections值，代表客户端至少被拒绝了一次，可以简单地设置为符合以下条件：Max_used_connections/max_connections=0.8

**6 Innodb 缓存命中率**

| 1234567891011 | mysql> show global status like 'innodb_buffer_pool_read%';+---------------------------------------+--------------+\| Variable_name       \| Value  \|+---------------------------------------+--------------+\| Innodb_buffer_pool_read_ahead_rnd  \| 0   \|\| Innodb_buffer_pool_read_ahead   \| 268720  \|  //预读的页数\| Innodb_buffer_pool_read_ahead_evicted \| 0   \|  \| Innodb_buffer_pool_read_requests  \| 480291074970 \| //从缓冲池中读取的次数\| Innodb_buffer_pool_reads    \| 29912739  \|   //表示从物理磁盘读取的页数+---------------------------------------+--------------+5 rows in set (0.00 sec) |
| ------------- | ------------------------------------------------------------ |
|               |                                                              |

缓冲池命中率 = (Innodb_buffer_pool_read_requests)/(Innodb_buffer_pool_read_requests + Innodb_buffer_pool_read_ahead + Innodb_buffer_pool_reads)=99.994%

如果该值小于99.9%，建议就应该增大innodb_buffer_pool_size的值了，该值一般设置为内存总大小的75%-85%，或者计算出操作系统所需缓存+mysql每个连接所需的内存（例如排序缓冲和临时表）+MyISAM键缓存，剩下的内存都给innodb_buffer_pool_size，不过也不宜设置太大，会造成内存的频繁交换，预热和关闭时间长等问题。

**7 MyISAM Key Buffer命中率和缓冲区使用率**

| 123456789101112131415161718192021222324252627282930 | mysql> show global status like 'key_%';+------------------------+-----------+\| Variable_name   \| Value  \|+------------------------+-----------+\| Key_blocks_not_flushed \| 0   \|\| Key_blocks_unused  \| 106662 \|\| Key_blocks_used  \| 107171 \|\| Key_read_requests  \| 883825678 \|\| Key_reads    \| 133294 \|\| Key_write_requests  \| 217310758 \|\| Key_writes    \| 2061054 \|+------------------------+-----------+7 rows in set (0.00 sec) mysql> show variables like '%key_cache_block_size%';+----------------------+-------+\| Variable_name  \| Value \|+----------------------+-------+\| key_cache_block_size \| 1024 \|+----------------------+-------+1 row in set (0.00 sec) mysql> show variables like '%key_buffer_size%';+-----------------+-----------+\| Variable_name \| Value  \|+-----------------+-----------+\| key_buffer_size \| 134217728 \|+-----------------+-----------+1 row in set (0.00 sec) |
| --------------------------------------------------- | ------------------------------------------------------------ |
|                                                     |                                                              |

缓冲区的使用率=1-(Key_blocks_unused*key_cache_block_size/ key_buffer_size)=18.6%

读命中率=1-Key_reads /Key_read_requests=99.98%

写命中率=1-Key_writes / Key_write_requests =99.05%

可看到缓冲区的使用率并不高，如果很长一段时间后还没有使用完所有的键缓冲，可以考虑把缓冲区调小一点。

键缓存命中率可能意义不大，因为它和应用相关，有些应用在95%的命中率下就工作良好，有些则需要99.99%，所以从经验上看，每秒的缓存未命中次数更重要，假设一个独立磁盘每秒能做100个随机读，那么每秒有5个缓冲未命中可能不会导致I/O繁忙，但每秒80个就可能出现问题。

每秒缓存未命中=Key_reads/uptime=0.33

**8 临时表使用情况**

| 1234567891011121314151617 | mysql> show global status like 'Created_tmp%';+-------------------------+----------+\| Variable_name   \| Value \|+-------------------------+----------+\| Created_tmp_disk_tables \| 19226325 \|\| Created_tmp_files  \| 117  \|\| Created_tmp_tables  \| 56265812 \|+-------------------------+----------+3 rows in set (0.00 sec) mysql> show variables like '%tmp_table_size%';+----------------+----------+\| Variable_name \| Value \|+----------------+----------+\| tmp_table_size \| 67108864 \|+----------------+----------+1 row in set (0.00 sec) |
| ------------------------- | ------------------------------------------------------------ |
|                           |                                                              |

可看到总共创建了56265812 张临时表，其中有19226325 张涉及到了磁盘IO，大概比例占到了0.34，证明数据库应用中排序，join语句涉及的数据量太大，需要优化SQL或者增大tmp_table_size的值，我设的是64M。该比值应该控制在0.2以内。

**9 binlog cache使用情况**

| 12345678910111213141516 | mysql> show status like 'Binlog_cache%'; +-----------------------+----------+\| Variable_name   \| Value \|+-----------------------+----------+\| Binlog_cache_disk_use \| 15  \|\| Binlog_cache_use  \| 95978256 \|+-----------------------+----------+2 rows in set (0.00 sec) mysql> show variables like 'binlog_cache_size';+-------------------+---------+\| Variable_name  \| Value \|+-------------------+---------+\| binlog_cache_size \| 1048576 \|+-------------------+---------+1 row in set (0.00 sec) |
| ----------------------- | ------------------------------------------------------------ |
|                         |                                                              |

Binlog_cache_disk_use表示因为我们binlog_cache_size设计的内存不足导致缓存二进制日志用到了临时文件的次数

Binlog_cache_use 表示 用binlog_cache_size缓存的次数

当对应的Binlog_cache_disk_use 值比较大的时候 我们可以考虑适当的调高 binlog_cache_size 对应的值

**10 Innodb log buffer size的大小设置**

| 1234567891011121314 | mysql> show variables like '%innodb_log_buffer_size%';+------------------------+---------+\| Variable_name   \| Value \|+------------------------+---------+\| innodb_log_buffer_size \| 8388608 \|+------------------------+---------+1 row in set (0.00 sec)mysql> show status like 'innodb_log_waits';+------------------+-------+\| Variable_name \| Value \|+------------------+-------+\| Innodb_log_waits \| 0  \|+------------------+-------+1 row in set (0.00 sec) |
| ------------------- | ------------------------------------------------------------ |
|                     |                                                              |

innodb_log_buffer_size我设置了8M，应该足够大了；Innodb_log_waits表示因log buffer不足导致等待的次数，如果该值不为0，可以适当增大innodb_log_buffer_size的值。

**11 表扫描情况判断**

| 12345678910111213 | mysql> show global status like 'Handler_read%';+-----------------------+--------------+\| Variable_name   \| Value  \|+-----------------------+--------------+\| Handler_read_first \| 19180695  \|\| Handler_read_key  \| 30303690598 \|\| Handler_read_last  \| 290721  \|\| Handler_read_next  \| 51169834260 \|\| Handler_read_prev  \| 1267528402 \|\| Handler_read_rnd  \| 219230406 \|\| Handler_read_rnd_next \| 344713226172 \|+-----------------------+--------------+7 rows in set (0.00 sec) |
| ----------------- | ------------------------------------------------------------ |
|                   |                                                              |

Handler_read_first：使用索引扫描的次数，该值大小说不清系统性能是好是坏

Handler_read_key：通过key进行查询的次数，该值越大证明系统性能越好

Handler_read_next：使用索引进行排序的次数

Handler_read_prev：此选项表明在进行索引扫描时，按照索引倒序从数据文件里取数据的次数，一般就是ORDER BY ... DESC

Handler_read_rnd：该值越大证明系统中有大量的没有使用索引进行排序的操作，或者join时没有使用到index

Handler_read_rnd_next：使用数据文件进行扫描的次数，该值越大证明有大量的全表扫描，或者合理地创建索引，没有很好地利用已经建立好的索引

**12 Innodb_buffer_pool_wait_free**

| 1234567 | mysql> show global status like 'Innodb_buffer_pool_wait_free';+------------------------------+-------+\| Variable_name    \| Value \|+------------------------------+-------+\| Innodb_buffer_pool_wait_free \| 0  \|+------------------------------+-------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

该值不为0表示buffer pool没有空闲的空间了，可能原因是innodb_buffer_pool_size设置太大，可以适当减少该值。

**13 join操作信息**

| 1234567 | mysql> show global status like 'select_full_join';+------------------+-------+\| Variable_name \| Value \|+------------------+-------+\| Select_full_join \| 10403 \|+------------------+-------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

该值表示在join操作中没有使用到索引的次数，值很大说明join语句写得很有问题

| 1234567 | mysql> show global status like 'select_range';+---------------+----------+\| Variable_name \| Value \|+---------------+----------+\| Select_range \| 22450380 \|+---------------+----------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

该值表示第一个表使用ranges的join数量，该值很大说明join写得没有问题，通常可查看select_full_join和select_range的比值来判断系统中join语句的性能情况

| 1234567 | mysql> show global status like 'Select_range_check';+--------------------+-------+\| Variable_name  \| Value \|+--------------------+-------+\| Select_range_check \| 0  \|+--------------------+-------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

如果该值不为0需要检查表的索引是否合理，表示在表n+1中重新评估表n中的每一行的索引是否开销最小所做的联接数，意味着表n+1对该联接而言并没有有用的索引。

| 1234567 | mysql> show GLOBAL status like 'select_scan';+---------------+-----------+\| Variable_name \| Value  \|+---------------+-----------+\| Select_scan \| 116037811 \|+---------------+-----------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

select_scan表示扫描第一张表的连接数目，如果第一张表中每行都参与联接，这样的结果并没有问题；如果你并不想要返回所有行但又没有使用到索引来查找到所需要的行，那么计数很大就很糟糕了。

**14 慢查询**

| 1234567 | mysql> show global status like 'Slow_queries';+---------------+--------+\| Variable_name \| Value \|+---------------+--------+\| Slow_queries \| 114111 \|+---------------+--------+1 row in set (0.00 sec) |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

该值表示mysql启动以来的慢查询个数，即执行时间超过long_query_time的次数，可根据Slow_queries/uptime的比值判断单位时间内的慢查询个数，进而判断系统的性能。

**15表锁信息**

| 12345678 | mysql> show global status like 'table_lock%';+-----------------------+------------+\| Variable_name   \| Value  \|+-----------------------+------------+\| Table_locks_immediate \| 1644917567 \|\| Table_locks_waited \| 53   \|+-----------------------+------------+2 rows in set (0.00 sec) |
| -------- | ------------------------------------------------------------ |
|          |                                                              |

这两个值的比值：Table_locks_waited /Table_locks_immediate 趋向于0，如果值比较大则表示系统的锁阻塞情况比较严重

**16查看当前运行的所以事务**

SELECT * FROM  information_schema.innodb_trx;

**17当前线程处理情况**

show FULL PROCESSLIST;

一般用到 show processlist 或 show full processlist 都是为了查看当前 mysql 是否有压力，都在跑什么语句，当前语句耗时多久了，有没有什么慢 SQL 正在执行之类的

可以看到总共有多少链接数，哪些线程有问题(time是执行秒数，时间长的就应该多注意了)，然后可以把有问题的线程 kill 掉，这样可以临时解决一些突发性的问题。

**18查看数据连接池配置信息**

show variables；

19查看数据库容量

select

table_schema as '数据库',

table_name as '表名',

table_rows as '记录数',

truncate(data_length/1024/1024, 2) as '数据容量(MB)',

truncate(index_length/1024/1024, 2) as '索引容量(MB)'

from information_schema.tables

where table_schema='ztocwst_sxyp'

order by data_length desc, index_length desc;