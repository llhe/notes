HotSpot JVM GC
=========

常用的GC算法
---------------
* reference counting
  * pros: simple，std::string，leveldb 数据结构，postgres bpool page PIN/UNPIN
  * cons: 每次引用的修改都需要inject额外指令，不能处理环形引用，GCer都不使用

* tracing collector
  stop-of-the-world collector, 从root引用开始追踪，找出所有有引用的对象
  root引用：寄存器中/栈上的局部变量，静态变量  
  * mark-sweep collector
    tracing collector的一种，trace并标记，最后sweep所有对象去掉未mark对象
      * pros：无需inject指令，无需编译器配合
      * cons：stop of the world，mark完后还需sweep一遍(需要访问garbage对象)，会产生大量碎片(cache locality, memory wastage)
  * copy collector
    tracing collector的另一种，两个工作区，tracing过程同时copy到另一个工作区
      * pros：只访问live object，cache locality好，无碎片，分配时可以简单bump-the-pointer高效
      * cons：占用空间，拷贝耗时
      * 注：都需要调用finalize()，mark-sweep需要在GC stop过程中调用？copy collector何时调用？finalize使用频率不高
  * Mark-compact collectors
     类似copy collector，但是不是copy，而是compact

Generational assumption
----------------------------
大多数应用的内存使用符合80-20准则，即大部分对象生存期较小。JVM采用划代内存管理，分为新生代(eden, S0, S1)，旧生代，以及permgen。

TLAB
-----------------
新生代中划分一块空间为每个线程提供分配，无锁，分配效率高(对于大对象或者用完时仍会在全局空间分配)。
`-XX:TLABWasteTargetPercent`: 占eden空间的百分比
`-XX:+PrintTLAB`: 打印TLAB使用情况

新生代可用GC
-----------------
* Serial GC: `-XX:+UseSerialGC`
  Minor GC时，应避免扫描整个旧生代采用remember set技术(card table)来标记。具体实现时，当对象某一个域赋值时，产生一个内存屏障，并检查引用是否是从旧生代到新生代的引用，如果是，更新card table。
  Minor GC时，仅扫描新生代，以及card table标记的旧生代对象，扫描起点是根对象(静态对象，常量对象，栈上对象--通过逃逸分析优化分配的，局部引用的对象)。

* Parallel Scavenge GC: `-XX:+UseParallelGC`
  Serial GC的并行版本(*并行*：多线程，仍然是stop-of-the-world; *并发*：避免stop-of-the-world，GC跟用户程序并发执行)。

* ParNew: `-XX:+UseParNewGC`
  并发版本，配合CMS

旧生代可用GC
-----------------
* Serial MSC: `-XX:+UseSerialGC`
  串行的标记，compaction
* Parallel Mark Sweep Parallel Compacting: `-XX:+UseParallelOldGC`
  并行的标记，compaction
* CMS: `-XX:+UseConcMarkSweepGC`
  并发版本：
    * 第一次标记：此阶段须停用应用
    * 并发标记
    * 重新标记：此阶段须停用应用
    * 并发收集

Full GC
--------------
某些情况会同时触发一种特殊的GC(新旧同时进行，细节有待整理)，典型情况是旧生代空间不足(如内存碎片，因为CMS并不做compaction)


