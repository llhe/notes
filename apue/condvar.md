condition variable如何signal或者broadcast
----------------------------------------
方案1：  
```C
pthread_mutex_lock(&mutex);
predicate=true;
pthread_cond_signal(&cv);
pthread_mutex_unlock(&mutex);
```
方案2:  
```C
pthread_mutex_lock(&mutex);
predicate=true;
pthread_mutex_unlock(&mutex);
pthread_cond_signal(&cv);
```

应该采用第一种，因为第二种先解锁，可能导致条件被第三方修改，被唤起的线程/进程发现条件不成立再次等待，影响性能。

但是第一种，signal时很可能会导致重新调度，也会影响性能，所以不少pthread的实现引入wait morphing的技术(包括NPTL实现):  
> Some Pthreads implementations address this deficiency using an optimization called wait morphing[1]. This optimization moves directly the threads from the condition variable queue to the mutex queue without context switch, when the mutex is locked. For instance, NPTL implements a similar technique to optimize the broadcast operation[2].
