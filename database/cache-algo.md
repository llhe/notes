cache/bpool/page replacement algorithm
========================

[Reference](http://en.wikipedia.org/wiki/Cache_algorithms)


Bélády's Algorithm
------------------
理想情况，通过事后分析用于对比不同算法，或者playback方式。

Time Based
------------------
根据过期时间进行淘汰

FIFO
------------------
没有明显局部性/记录过期时间相同，且性能不是瓶颈的情况，比如客户端

Second-chance
-------------------
FIFO的变形形式，维持FIFO队列，依次检查usage bit，如果设置，则清零，并放到队尾

Clock sweep
------------------
* LRU的缺点是节点链表在读取时均需要加锁，且只考虑了recency
* 比Second-chance实现效率高，不需要每次重新插入队尾，且可以定制recency与frequency的比重(即usage count的最大值，postgres的`BM_MAX_USAGE_COUNT`变量)

Clock-Pro
-----------------

MRU
------------------
适合非正常场景，例如重复的大表全表扫描，随机访问，比LRU效果好

LFU
------------------
只根据频率，需要客户端主动删除数据(或者设定超时时间)，记忆性太好所以适应性差

LRU
------------------
实现方式:
* (Sharded) hash table
* double linked list
变体包括LRU-K/2Q/MQ等



LRU-K (LRU-2 iff K=2)
------------------
维护一个临时队列记录访问历史，不记录数据本身(采用FIFO或者LRU)，仅当此队列中的对象被访问达到K次时才加入到主LRU队列。一般使用LRU2

Two Queue (2Q)
------------------
* 维护一个较小的FIFO队列和一个较大的LRU队列，FIFO用于插入新对象，第二次访问时放入LRU队列，两者大小维持在1:3左右(应该可以调节)，性能优于LRU2
* 第一个队列对应least recency，第二个对应least frequency (因为是2次)，ARC类似
* 与LRU2的区别是LRU2第一个队列(显然也是FIFO的，因为第二次即被移出)仅保存访问历史，不保存数据本身

Multiple Queue (MQ)
------------------
维护多个LRU队列，且有不同优先级，数据会有提升，降低优先级的操作在队列中移动。缺点是复杂度太高。


ARC (Adaptive Replacement Cache)
------------------
2Q的变体，每个队列底部外接一个ghost队列，只存储元数据，不存储值(类似LRU-K)，算法自适应，规则见[wiki](http://en.wikipedia.org/wiki/Adaptive_replacement_cache)
IBM专利，IBM存储控制器，ZFS有应用，Postgres 8.0曾使用过

LIRS
-------------------
MySQL NDB使用


Case study
-------------------
* Memcached: LRU
* [postgres](https://github.com/postgres/postgres/tree/master/src/backend/storage/buffer) 
    * 使用类clock sweep算法 (截止到9.3beta，8.0曾短暂使用ARC算法)：每个page，有一个count (上限为`BM_MAX_USAGE_COUNT`，默认为5，即如果最近访问五次，有五次机会survive)，每次`PIN (BuﬀerAlloc)`的时候+1，每次扫描-1，扫描时为0(前提是未被正在使用的，ping/ref count 为0)则选为victim而被evict。
    * 对于`VACUUM`和`seq scan`单独分配ring buffer，采用相同的clock sweep算法，buffer ring大小为256k，目的是大于CPU的L2 cache line (For sequential scans, a 256KB ring is used. That's small enough to fit in L2 cache, which makes transferring pages from OS cache to shared buffer cache efficient.  Even less would often be enough, but the ring must be big enough  to accommodate all pages in the scan that are pinned concurrently.)
* InnoDB:  
LRU 变体： http://dev.mysql.com/doc/refman/5.5/en/innodb-buffer-pool.html
另外，InnoDB对于压缩表(类似LevelDB)，分别cache压缩数据和解压后的数据，根据系统的IO/CPU bound类型，自适应调整LRU和unzip_LRU队列。
* HBase:  
2Q变体
* Linux Kernel
