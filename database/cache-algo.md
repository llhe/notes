cache/bpool/page replacement algorithm
========================

[Reference](http://en.wikipedia.org/wiki/Cache_algorithms)
http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/io/hfile/BlockCache.html


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
