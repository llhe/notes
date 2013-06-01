leveldb的写入流程
=====================
leveldb的写入操作(Put, Delete, Batch Write)最终都是通过Write调用来实现。多线程并发写入时，(可能)需要sync WAL，为避免随机IO，leveldb采用了group commit(但是没有延时)，实现方式类似leader-follower方式(不完全相同，sync既是accept过程也是serve过程)。

```C++
// Information kept for every waiting writer
struct DBImpl::Writer {
  Status status;
  WriteBatch* batch;
  bool sync;
  bool done;
  port::CondVar cv;

  explicit Writer(port::Mutex* mu) : cv(mu) { }
};
```

```C++
std::deque<Writer*> writers_; // 由全局的mutex_保护
```

```C++
  Status DBImpl::Write( const WriteOptions& options, WriteBatch* my_batch) {
  Writer w(&mutex_);
  w.batch = my_batch;
  w.sync = options.sync;
  w.done = false ;

  MutexLock l(&mutex_);
  writers_.push_back(&w); // // 每个写入线程在写入时初始化一个writer句柄w，并插入到全局队列writers_中
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status; // 已经被其他group commit写入完成
  }

  // 此时成为leader，取得当前writers_列表以及sequence number快照
  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(my_batch == NULL ); // XXX
  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  if (status.ok() && my_batch != NULL ) {  // NULL batch is for compactions
    WriteBatch* updates = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(updates, last_sequence + 1 );
    last_sequence += WriteBatchInternal::Count(updates);

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      mutex_.Unlock();
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));
      if (status.ok() && options.sync) {
        status = logfile_->Sync(); // 注意，非sync的写线程不顺带写入sync=true的写入，见BuildBatchGroup实现
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(updates, mem_);
      }
      mutex_.Lock();
    }
    if (updates == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }

  // 标记并唤醒本批次处理完成的writer以及下一个需要处理的follower作为leader
  while ( true ) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true ;
      ready->cv.Signal();
    }
    if (ready == last_writer) break ;
  }

  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

```C++
WriteBatch* DBImpl::BuildBatchGroup(Writer** last_writer) {
// ...
   for (; iter != writers_.end(); ++iter) {
    Writer* w = *iter;
    if (w->sync && !first->sync) {
      // Do not include a sync write into a batch handled by a non-sync write.
      break ;
    }
 // ...
 }
```

其中`MakeRoomForWrite`可能阻塞写线程:
```
// level 0文件个数达到上限，阻塞1毫秒
versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger
```
```
// memtable超过限额且immutable memtable还未dump完成，等待直到完成 bg_cv_.Wait();
mem_->ApproximateMemoryUsage() > options_.write_buffer_size && imm_ != NULL
```
```
// level 0文件个数达到kL0_StopWritesTrigger上限，等待bg_cv_.Wait();
mem_->ApproximateMemoryUsage() > options_.write_buffer_size &&
versions_->NumLevelFiles( 0) >= config::kL0_StopWritesTrigger
```
MakeRoomForWrite代码如下：
```C++
// REQUIRES: mutex_ is held
// REQUIRES: this thread is currently at the front of the writer queue
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;
  while (true) {
    if (!bg_error_.ok()) {
      // Yield previous error
      s = bg_error_;
      break;
    } else if (
        allow_delay &&
        versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;  // Do not delay a single write more than once
      mutex_.Lock();
    } else if (!force &&
               (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
      // There is room in current memtable
      break;
    } else if (imm_ != NULL) {
      // We have filled up the current memtable, but the previous
      // one is still being compacted, so we wait.
      Log(options_.info_log, "Current memtable full; waiting...\n");
      bg_cv_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      // There are too many level-0 files.
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      bg_cv_.Wait();
    } else {
      // Attempt to switch to a new memtable and trigger compaction of old
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = NULL;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
      if (!s.ok()) {
        // Avoid chewing through file number space in a tight loop.
        versions_->ReuseFileNumber(new_log_number);
        break;
      }
      delete log_;
      delete logfile_;
      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);
      imm_ = mem_;
      has_imm_.Release_Store(imm_);
      mem_ = new MemTable(internal_comparator_);
      mem_->Ref();
      force = false;   // Do not force another compaction if have room
      MaybeScheduleCompaction();
    }
  }
  return s;
}
```
