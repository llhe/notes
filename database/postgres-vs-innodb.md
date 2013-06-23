Postgres v.s. MySQL/InnoDB
==========================

Index
----------
1. InnoDB
  * index organized table，B+树叶子为数据
  * 非唯一二级索引有insert/update buffer避免随机读

2. pg
  * 索引，堆文件分离，B+树索引，但跟InnoDB没有太大本质区别(包括IO性能)
  * pg无insert buffer优化
  
Cache替换算法
------------
1. InnoDB
  * 2Q变体
2. pg
  * clock sweep (N次机会，N为最近访问次数，默认N最大值为5)

How to handle torn page
-----------
注意，如果底层文件系统(zfs支持checksum)或存储故障(bit rot，硬件bug等)，不一定可恢复(从这一角度，相对而言pg更安全)
1. InnoDB  
  * double write buffer：一般仅适用于half write引起的page corrupt
  
2. pg
  * [full page write](http://www.xaprb.com/blog/2010/02/08/how-postgresql-protects-against-partial-page-writes-and-data-corruption/) (configurable) in WAL when first write after previous checkpoint

How to detect page corruption
----------
1. InnoDB
  * page checksum

2. pg
  * WAL页有CRC32 checksum，而数据页没有(vPostgres有过此patch，[9.3Beta 1加入可此feature](http://wiki.postgresql.org/wiki/What's_new_in_PostgreSQL_9.3))

Compression
----------
1. InnoDB
  compression table: bpool根据cpu/io负载自适应保存压缩/解压的页面 (压缩后的页面如何对齐？每一个压缩页(1k,2k,4k,8k,16k)都对应了一个16k的非压缩页，修改后，尝试压缩，是否重新分配页面获取新的page id更新索引？)

2. pg
  * only large field (> page size/4 for specific types) will be compressed (and chuncked with TOAST, to reduce IO since large field is accessed infrequently)
  * can leverage underlying file system compression, e.g. ZFS

MVCC
------------
1. pg
  * 事务开始时，记录活跃事务列表(SnapshotData)
  * pg_clog(txn commit log, pg_xlog目录是WAL)记录历史事务(最终)状态：已提交，已回滚，活跃 (InnoDB无需保存此表，因为记录状态只有已提交和活跃两种，即一个事务开始时已回滚的事务的修改已经物理删除掉了，所以通过活跃事务快照即可判断可见性。题外话，percolator需要两个时间戳，一个是第一次读的时间戳，另一个是提交的时间戳，实际上第二个时间戳作用等同于pg_clog)
  * tuple中每个版本有xmin，xmax分别记录创建，删除(或覆盖)其的事务ID，所有版本串成链表
  * 根据MVCC规则和当前隔离级别判断是否可见

MVCC 历史版本
------------
两者最大的区别在于历史版本存储是否分离。  
pg回滚不需要做额外操作，只需设置pg_clog对应状态为已回滚，而InnoDB则需回滚操作。

1. InnoDB
  * put any old versions into UNDO log (in rollback segment tablespace), even uncommited changes (the rationale is the rollback is much rare)
  * InnoDB主键包含事务信息：主键索引记录的头上包含有6字节的事务ID(未必对当前事务可见，甚至可能未提交，但仍有很大作用)与7字节指向回滚段中旧版本的指针
  * 二级索引项本身不包含版本可见信息，但每个页面头包含对其修改的最大事务ID(对于大多数冷页面应该是肯定可见的)，如果肯定可见，则直接返回属性，不去读取堆
  * 主键索引，二级索引均包含历史版本(即可能已经被删除，但由于MVCC，不能立即删除，需要后续purge)
  * 回滚段，主索引，二级索引需要purge线程来清理，类似VACUUM

2. pg
  * Postgres keep all history in-place(同InnoDB，索引和堆都包含历史版本，需要进行清理), then no UNDO log is needed. It relies on VACUUM to recycle the space
  * index 不包含事务可见性信息，所以`select count(*)` 性能低下(InnoDB有一样的问题，但相对来讲，其heap中存储的过时信息少，IO效率可能高一些)
  * visibility map：每个relation对应一个vm (同样有一个fsm，以二叉树形式保存可用空间)，每个bit对应一个物理页面 (*The visibility map is conservative in that a set bit (1) indicates that all tuples are visible on the page, but an unset bit (0) indicates that that condition may or may not be true*，即为1的话，页面上所有tuple对所有活动事务都是可见的)。9.2之前，仅用于加速vacuum，即不去访问1的页面，9.2引入的index only scan使用此信息避免访问heap。

vPostgres paches
-------
1. memory ballooning
  * hypervisor -- balloon driver -- pg buffer manager 之间互相通信，hypervisor和pg buffer driver动态调整内存使用，balloon driver进行两者之间的内存转移

2. page checksum  
  * 9.3将引入此特性，貌似没采用vmw的patch，dev都跑路了

3. direct io
