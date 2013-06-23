Postgres v.s. Google percolator v.s. Alta v.s. Yahoo omid
============================================

PostgreSQL
----------------
0. 相关模块/数据结构
  * 活跃事务快照`SnapshotData` (结构中保存`xmin`，`xmax`分别保证`xid < xmin`肯定可见，`xid > xmax`肯定不可见，减少查找次数)
  * pg_clog历史事务状态日志：已提交，已回滚，活跃 (InnoDB无需此信息，因为abort的物理记录清理是eager方式)
  * tuple中保存xmin，xmax (类似，为实现subtransaction、savepoint还有额外cid信息)
  * 自增长xid生成`GetNewTransactionId`
1. 事务规则
  * 事务开始时获取xid和SnapshotData
  * 读取时根据隔离级别判断可见性(略)
  * 提交时检测`写-写`冲突(9.0实现了Serializable Snapshot Isolation，是在SI基础上实现检测write skew，不同于InnoDB的naive两阶段锁)

Percolator
----------------
1. 事务规则
  * 第一次读取获取时间戳T1，所有后续读可见性根据T1
  * 所有写入缓存在客户端
  * 两阶段提交
    * prepare 阶段
	  * 选取一个记录为主记录(如第一个写入的)
	  * 写入所有记录，每个记录的lock域指向主记录，每个记录的写入为行事务原子操作
	* commit 阶段
	  * 获取时间戳T2
	  * commit 主记录，清空lock，更新write域为对应的T2，单行事务原子操作
	  * commit其他记录

2. 可读性与清理(redo or undo)
  * 没有全局TM，清理工作由客户端完成(遇到crash由后续客户端处理)
  * 对于读取含有lock的记录，先尝试清理(可能阻塞)，然后根据可见性可返回对应版本
  * `BackoffAndMaybeCleanupLock`的实现?
      * 如果是通过chubby检测可见性的话，是否需要在主记录存储一个事务拥有者的UUID/GUID？应该需要，开销：时间8B+机器xB+随机数yB=zB
      * 由于当事务拥有者还活着的话，此函数可能会等待，但正常情况不commit不会太久，但是chubby的lease周期不会太短(但这不是critical/normal path)
	
Alta California
-----------------
0. [logical timestamp](http://en.wikipedia.org/wiki/Lamport_timestamps)
  * A process increments its counter before each event in that process;
  * When a process sends a message, it includes its counter value with the message;
  * On receiving a message, the receiver process sets its counter to be greater than the maximum of its own value and the received value before it considers the message received.
  
1. 事务规则
  * 第一次读取获取时间戳`RSN`，所有后续读可见性根据`RSN`
  * 所有写入缓存在客户端
  * 两阶段提交
    * prepare 阶段
	  * 选取第一个写入的记录为主记录，写入，然后返回`CSN0`
	  * 携带`CSN0`写入其余记录返回`CSNi >= CSN0`：每个记录的lock域指向主记录，每个记录的写入为行事务原子操作
	* commit 阶段
	  * commit 主记录返回最终`CSN >= max(CSN0...i)`，清空lock，更新write域为对应的`CSN`，单行事务原子操作
	  * commit其他记录，写入`write=CSN`

2. Snapshot Isolation Read细节
  * 如果记录过期被垃圾回收，则读取返回空
  * 如果读取时，tablet时钟小于读事务的`RSN`，则读取阻塞或被拒绝(读取操作不会导致clock更新？why？会导致何种问题？)
  * 如果读取遇到处在prepare阶段的记录，则
      * 等待事务结束
      * 超时则尝试resolve对应的事务(如果已提交则读取，返回，否则类似google chuuby存储进程状态？实际上可以单纯的更新其对应主记录对应tablet的时钟，然后返回旧版本)
      * 上一步超时则拒绝请求
  * 同一个客户端在一个事务中写入一个记录提交后在下一个事务可能读不到此记录(因为第二个事务获取的CSN可能来自另外一个不相关的tablet)，解决办法是客户端若要保证因果关系，需要遵守lamport clock规则，即在第二个事务请求携带前一事务的`CSN`

3. 时钟同步：单主节点UDP广播

4. 额外特性
  * pubsub
  * Tally Table: 读取为read commited，写入(ADD操作)先缓存，然后参与2PC的prepare


Yahoo Omid
-----------------
1. 事务规则
  * start  
    client 获取StartTS，选择性piggyback TSO缓存(活动事务快照，事务执行过程中提交之前不再需要跟TSO通信)
  * write row  
    写入StartTS
  * read row  
    需要获取所有版本 (或者可以使用HBase filter？)，然后对比本地txn表判断可见性 (只需要本地缓存事务开始时的TSO全局状态即可，因为以后提交的一定不可见)
  * commit  
    向TSO请求，成功更新记录CommitTS(无需返回给client)，失败需要清理  
    类似于InnoDB，失败立即清理物理记录，然后TSO不再保存其事务状态(因为物理记录中找不到此事务的任何信息)，否则需要类似pg_clog信息，对于TSO来说开销太大
  * clean up  
    删除对应版本

2. 细节解释
  * TSO包含以下信息(Transactional Metadata，即复制到client的信息，开销较大，wiki声称支持1000个client)
  * 活跃或已提交事务列表(`StartTS`, `CommitTS`)
	 * 已abort事务列表
	 * `Row LastCommitTS`  
	  记录近期所有更新的记录对应的最后更新的CommitTS，因为TSO做`写-写`冲突仲裁，所以需要此信息(注：此处可能为系统瓶颈，每次提交需要传输、更新、保存所有修改记录，需要内存中存储，并记录WAL/bookkeeper)
  * `LowWatermark`  
	  为限制当前活跃事务数目(前三个表的大小跟此值相关)，维护一个持续增长的txn low wartermark，强制abort超出的长事务
  * 与ReTSO paper介绍的不大相同
      * [github code 对应的clides](http://yahoo.github.io/omid/docs/hadoop-summit-europe-2013.pdf)
      * paper上讲如果commit成功也会进行清理，会改写CommitTS以替换StartTS。且RegionServer会区分未提交的写入，好处是不用每个client都在开始时对当前所有未结束事务列表做快照(以处理 TSs_j < TSs_i < TSc_j < TSc_i 的读可见性问题，应该判定为不可见，如果不做快照则不能解决，类似PostgreSQL实现)

3. 优缺点
  * 优点
     * 实现简单，与单机TM基本一致
     * 无需加锁(主要区别是client crash导致get阻塞，正常情况区别不大)，因为获取时间戳和提交事务都在TSO，是原子操作
  * 缺点
     * TSO SPOF，且与客户端通信开销大
     * 应该不太适合在线系统，yahoo声称用于个性化推荐系统
