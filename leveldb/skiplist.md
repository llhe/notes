leveldb memtable
==================

leveldb的memtable采用skiplist实现。

并发控制
--------------------------
* 插入操作需要外部加锁
* 读操作不需要加锁，但需保证节点再读时不被删除
  原理：
    ** 插入时再更新前置指针前插入内存屏障(修改新节点时不需要以提高性能，类似于log group sync)
    ** 读取指针前插入内存屏障，强制刷新CPU Cache

```C++
// Thread safety
// -------------
//
// Writes require external synchronization, most likely a mutex.
// Reads require a guarantee that the SkipList will not be destroyed
// while the read is in progress.  Apart from that, reads progress
// without any internal locking or synchronization.
//
// Invariants:
//
// (1) Allocated nodes are never deleted until the SkipList is
// destroyed.  This is trivially guaranteed by the code since we
// never delete any skip list nodes.
//
// (2) The contents of a Node except for the next/prev pointers are
// immutable after the Node has been linked into the SkipList.
// Only Insert() modifies the list, and it is careful to initialize
// a node and use release-stores to publish the nodes in one or
// more lists.
//
// ... prev vs. next pointer ordering ...
```

GCC memory clobber
```C++
// Gcc on x86
#elif defined(ARCH_CPU_X86_FAMILY) && defined(__GNUC__)
inline void MemoryBarrier() {
  // See http://gcc.gnu.org/ml/gcc/2003-04/msg01180.html for a discussion on
  // this idiom. Also see http://en.wikipedia.org/wiki/Memory_ordering.
  __asm__ __volatile__( "" : : : "memory" );
}
#define LEVELDB_HAVE_MEMORY_BARRIER
```

原子指针(除了内存屏障之外，是否还需要赋值操作在一个指令内完成？内存必须对齐？如果AtomicPointer通过malloc分配或者编译器在栈上分配则没有问题，但是强制reinterpret_cast一段内存时危险的[call for reference])
```C++
// AtomicPointer built using platform-specific MemoryBarrier()
#if defined(LEVELDB_HAVE_MEMORY_BARRIER)
class AtomicPointer {
 private:
  void* rep_;
 public:
  AtomicPointer() { }
  explicit AtomicPointer( void* p) : rep_(p) {}
  inline void* NoBarrier_Load() const { return rep_; }
  inline void NoBarrier_Store(void * v) { rep_ = v; }
  inline void* Acquire_Load() const {
    void* result = rep_;
    MemoryBarrier(); // 读取变量前更新缓存
    return result;
  }
  inline void Release_Store(void * v) {
    MemoryBarrier(); // 注意执行顺序，修改变量前强制把Cache中脏数据写到内存
    rep_ = v;
  }
};
```

插入时更新前置指针后通过生成的(x86)mfence指令刷cache
(_注意_，`Release_Store`的`MemoryBarrier()`是在赋值之前调用，确保之前的修改写入内存，而非赋值操作本身，因为两者之间可能会发生中断)
```C++
  void SetNext( int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.
    next_[n].Release_Store(x);
  }
```

插入节点
```C++
template<typename Key, class Comparator>
void SkipList<Key,Comparator>::Insert(const Key& key) {
  // TODO (opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
  Node* prev[kMaxHeight];
  Node* x = FindGreaterOrEqual(key, prev);

  // Our data structure does not allow duplicate insertion
  assert(x == NULL || !Equal(key, x->key));

  int height = RandomHeight();
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    //fprintf(stderr, "Change height from %d to %d\n", max_height_, height);

    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (NULL), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since NULL sorts after all
    // keys.  In the latter case the reader will use the new node.
    max_height_.NoBarrier_Store( reinterpret_cast<void *>(height)); 
  }

  x = NewNode(key, height);
  for ( int i = 0 ; i < height; i++) {
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i)); // 尽可能优化，减少mb的使用
    prev[i]->SetNext(i, x);
  }
}
```

声称新节点，节点高度的概率为0.25^n(而非二叉树等价的0.5^2，理论依据？)
```C++
template<typename Key, class Comparator>
int SkipList<Key,Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

使用`GetMaxHeight()`而非最大值`enum { kMaxHeight = 12 };`是为了提高效率(对于较小的memtable，实际最大高度往往到不了，使用`kMaxHeight`会多很多无效判断)。`GetMaxHeight()` 的读取不需要内存屏障，读取到过时数据不影响正确性(仅稍微影响查询速度，少skip一些)

```C++
  // Modified only by Insert().  Read racily by readers, but stale
  // values are ok.
  port::AtomicPointer max_height_;   // Height of the entire list

  inline int GetMaxHeight() const {
    return static_cast <int>(
        reinterpret_cast<intptr_t >(max_height_.NoBarrier_Load()));
  }
```

Why not lock free for insert?
---------------------
插入需要更新多个前置指针，所以没有办法通过CAS原子操作来实现(可能会丢掉并发插入)，lock-free实现：
* [stack overflow thread](http://stackoverflow.com/questions/3479043/how-to-implement-lock-free-skip-list)
* google it

Lesson Learned
---------------------
* 内存屏障的正确性
* 尽可能减少内存屏障的使用，以提高性能
* mutex 实现必须有内存屏障(x86可以是mfence系列指令,也可以是xchg, lock prefix等指令，不同CPU支持程度不同)，否则不肯能实现线程安全([Threads Cannot be Implemented as a Library](http://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf))
* 内存顺序需要编译器(指令级别优化，使用寄存器)和CPU(使用缓存，乱序执行等)共同配合，所以X86/X64虽无乱序，但也需要对应指令
* Java volatile有内存顺序保证(有mb)，可用于mt，比如实现一些lock-free结构
