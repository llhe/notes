Java同步机制
==============

同步机制
------
1. 语言级支持，JVM实现
  * synchronized (monitor)
     * 有一系列的优化(esp JIT)，比如转化成自旋锁，偏向锁优化(即锁的拥有线程重复加锁时，免去CAS操作，只记录一个计数器)
  * wait/signal
     * 一般由条件变量实现
  * volatile
  * final
  * Unsafe CAS
  * Unsafe park/unpark
     * park总会调用pthread_cond_wait/timewait->futex(除非前面有unpark先调用)，但monitor可能会优化成spinlock
	 * 不同于条件变量需要mutex配合，park采用的解决方法是记录上次未配对的unpark
	      * *Block current thread, returning when a balancing unpark occurs, or a balancing unpark has already occurred, 
	 or the thread is interrupted, or, if not absolute and time is not zero, the given time nanoseconds 
have elapsed, or if absolute, the given deadline in milliseconds since Epoch has passed, or spuriously (i.e., 
returning for no "reason"). Note: This operation is in the Unsafe class only because unpark is, so it would be 
strange to place it elsewhere.*
2. Unsafe实现的显式锁以及并发库
  * java.util.concurrent.locks 显式锁，用户可以扩展
     * 由Daug Lea实现，也实现了偏向锁，相对monitor的好处是灵活，且用户可扩展定义自己的锁
  * 其他并发容器，组件等

java 显式锁的实现(AbstractQueuedSynchronizer.java)
-------------------------
由于其实现回最终调用pthread_cond_wait(or timewait)，但是却没有使用系统的mutex(park/unpark有但是没暴露到外面)做保护(cont wait与mutex释放做不到原子操作)，所以存在的问题是如何使得park的线程不会饿死(即unlock的线程调用unpark时可能park尚未被调用)。
```Java
/**
* Wakes up node's successor, if one exists.
*
* @param node the node
*/
    private void unparkSuccessor(Node node) {
        /*
* If status is negative (i.e., possibly needing signal) try
* to clear in anticipation of signalling. It is OK if this
* fails or if status is changed by waiting thread.
*/
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

/*
* Thread to unpark is held in successor, which is normally
* just the next node. But if cancelled or apparently null,
* traverse backwards from tail to find the actual
* non-cancelled successor.
*/
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }

    /**
* Convenience method to park and then check if interrupted
*
* @return {@code true} if interrupted
*/
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

/*
* Various flavors of acquire, varying in exclusive/shared and
* control modes. Each is mostly the same, but annoyingly
* different. Only a little bit of factoring is possible due to
* interactions of exception mechanics (including ensuring that we
* cancel if tryAcquire throws exception) and other control, at
* least not without hurting performance too much.
*/

/**
* Acquires in exclusive uninterruptible mode for thread already in
* queue. Used by condition wait methods as well as acquire.
*
* @param node the node
* @param arg the acquire argument
* @return {@code true} if interrupted while waiting
*/
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
解决问题的关键是unpark有一次记忆功能，即如果调用unpark时，被唤醒的线程处于running状态(jvm自己保存的状态)，则标记一个特殊位`THREAD_PARK_PERMIT`，而后续的park会直接返回，不去调用pthread_cond_wait，所以不会出现忘记唤醒的情况发生。
```C++
/**
* Releases the block on a thread created by _Jv_ThreadPark().  This
* method can also be used to terminate a blockage caused by a prior
* call to park.  This operation is unsafe, as the thread must be
* guaranteed to be live.
*
* @param thread the thread to unblock.
*/
void
ParkHelper::unpark ()
{
  using namespace ::java::lang;
  volatile obj_addr_t *ptr = &permit;


  /* If this thread is in state RUNNING, give it a permit and return
     immediately.  */
  if (compare_and_swap
      (ptr, Thread::THREAD_PARK_RUNNING, Thread::THREAD_PARK_PERMIT))
    return;


  /* If this thread is parked, put it into state RUNNING and send it a
     signal.  */
  if (compare_and_swap
      (ptr, Thread::THREAD_PARK_PARKED, Thread::THREAD_PARK_RUNNING))
    {
      int result;
      pthread_mutex_lock (&mutex);
      result = pthread_cond_signal (&cond);
      pthread_mutex_unlock (&mutex);
      JvAssert (result == 0);
    }
}

/**
* Blocks the thread until a matching _Jv_ThreadUnpark() occurs, the
* thread is interrupted or the optional timeout expires.  If an
* unpark call has already occurred, this also counts.  A timeout
* value of zero is defined as no timeout.  When isAbsolute is true,
* the timeout is in milliseconds relative to the epoch.  Otherwise,
* the value is the number of nanoseconds which must occur before
* timeout.  This call may also return spuriously (i.e.  for no
* apparent reason).
*
* @param isAbsolute true if the timeout is specified in milliseconds from
*                   the epoch.
* @param time either the number of nanoseconds to wait, or a time in
*             milliseconds from the epoch to wait for.
*/
void
ParkHelper::park (jboolean isAbsolute, jlong time)
{
  using namespace ::java::lang;
  volatile obj_addr_t *ptr = &permit;


  /* If we have a permit, return immediately.  */
  if (compare_and_swap
      (ptr, Thread::THREAD_PARK_PERMIT, Thread::THREAD_PARK_RUNNING))
    return;


  struct timespec ts;


  if (time)
    {
      unsigned long long seconds;
      unsigned long usec;


      if (isAbsolute)
     {
       ts.tv_sec = time / 1000;
       ts.tv_nsec = (time % 1000) * 1000 * 1000;
     }
      else
     {
       // Calculate the abstime corresponding to the timeout.
       jlong nanos = time;
       jlong millis = 0;


       // For better accuracy, should use pthread_condattr_setclock
       // and clock_gettime.
#ifdef HAVE_GETTIMEOFDAY
       timeval tv;
       gettimeofday (&tv, NULL);
       usec = tv.tv_usec;
       seconds = tv.tv_sec;
#else
       unsigned long long startTime
         = java::lang::System::currentTimeMillis();
       seconds = startTime / 1000;
       /* Assume we're about half-way through this millisecond.  */
       usec = (startTime % 1000) * 1000 + 500;
#endif
       /* These next two statements cannot overflow.  */
       usec += nanos / 1000;
       usec += (millis % 1000) * 1000;
       /* These two statements could overflow only if tv.tv_sec was
          insanely large.  */
       seconds += millis / 1000;
       seconds += usec / 1000000;


       ts.tv_sec = seconds;
       if (ts.tv_sec < 0 || (unsigned long long)ts.tv_sec != seconds)
         {
           // We treat a timeout that won't fit into a struct timespec
           // as a wait forever.
           millis = nanos = 0;
         }
       else
         /* This next statement also cannot overflow.  */
         ts.tv_nsec = (usec % 1000000) * 1000 + (nanos % 1000);
     }
    }


  pthread_mutex_lock (&mutex);
  if (compare_and_swap
      (ptr, Thread::THREAD_PARK_RUNNING, Thread::THREAD_PARK_PARKED))
    {
      int result = 0;


      if (! time)
     result = pthread_cond_wait (&cond, &mutex);
      else
     result = pthread_cond_timedwait (&cond, &mutex, &ts);


      JvAssert (result == 0 || result == ETIMEDOUT);


      /* If we were unparked by some other thread, this will already
     be in state THREAD_PARK_RUNNING.  If we timed out or were
     interrupted, we have to do it ourself.  */
      permit = Thread::THREAD_PARK_RUNNING;
    }
  pthread_mutex_unlock (&mutex);
}
```

java thread state
------------
1. NEW 
已创建但未开始执行
2. RUNNABLE 
可执行状态，可以是正在执行，也可以是等待OS调度
3. BLOCKED
阻塞在monitor lock (Object.wait返回时亦可以阻塞)，显式Lock的阻塞出于什么状态？
4. WAITING 
等待其他事件(signal, signalAll, 其他线程结束) 
  * Object.wait with no timeout
  * Thread.join with no timeout
  * LockSupport.park
5. TIMED_WAITING 
有时间上限 
  * Thread.sleep
  * Object.wait with timeout
  * Thread.join with timeout
  * LockSupport.parkNanos
  * LockSupport.parkUntil
6. TERMINATED
