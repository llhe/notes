Cluster scheduler
==================

monolithic scheduler
--------------------
类似于MapReduce的JobTracker，固定调度一种任务  
缺点：
增加新的种类困难，或者说不可能

Tow-level scheduler
--------------------
YARN/GS, Mesos  
master负责管理调度分派全局资源，具体的任务(map-reduce, event processer host, and long run service like tablet/region     server)自己实现调度器/执行器(Yarn/Gs抽象成了run-command?)，具体的调度器分批(不需要一次性)从全局调度器申请资源，然后自己调度(包含启动long run node)具体的任务(short time or long run task/service)  
缺点：不能在全局进行资源均衡  
  * 子/第二级调度器不知道全局资源使用情况，不能做出最优调度
  * 分派给子/第二级调度器后的资源相当于被加了悲观锁，可能会造成浪费

Shared state scheduler
--------------------
所有调度器(多个)共享全局资源状态，使用乐观锁以及MVCC类似机制，并发执行调度 (经测试，使用率带来的资源利用率提升大于冲突造成的开销)
有一份master信息，各个调度器定期做一个本地拷贝，在本地拷贝基础上计算调度算法，计算完成后commit到master上，如果无冲突则提交成功，否则重试

问题：如何保证locality
  1. Cosmos GS: max flow/min-cut problem --> Ford–Fulkerson algorithm.
    finding a feasible flow through a single-source, single-sink flow network that is maximum.
  2. Mesos: use delay scheduling for data locality
  3. YARN: (YARN-80 Support delay scheduling for node locality)


Borg
--------------
Borg = Autopilot + YARN/Mesos/Omega/GS/Facebook Corona?


