OCC v.s. PCC
===============

严格意义上悲观锁，乐观锁说法不严谨，只是两种并发控制机制，不限定RDBMS。一般来说


* 悲观并发控制机制(PCC: pessimistic concurrency control)：在开始操作前锁定资源，适合并发冲突严重的情况

  Case study:
    1. Hibernate的悲观锁对应于`select * from tablename for update`
    2. RDBMS通过2PL实现不同隔离级别


* 乐观并发控制机制(OCC: optimistic concurrency control)：在最后提交时再检查是否有冲突，适合并发冲突较少的情况

  Case study:
    1. Hibernate 通过version/timestamp实现应用层OCC
    2. RDBMS的MVCC机制实现snapshot isolation (postgres, percolator, omid, baja, innodb)
    3. MediaWiki, Bugzilla的并发编辑都是应用层实现的OCC
    4. 某些lock free数据结构的实现可以看作是一种OCC (CAS success ? commit done : abort and retry)
    5. [vector clock](http://en.wikipedia.org/wiki/Vector_clocks)

