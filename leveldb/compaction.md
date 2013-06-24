leveldb compaction
===================

检查是否compact的时机(call MaybeScheduleCompaction)
----------------
* 打开db，recover完成后
* 写入时调用MakeRoomForWrite检查memtable占用空间是否达到阈值
* 读时检查allowed_seeks次数是否超限

```C++
if (have_stat_update && current->UpdateStats(stats)) {
   MaybeScheduleCompaction();
}
```
```C++
bool Version::UpdateStats(const GetStats& stats) {
  FileMetaData* f = stats.seek_file;
  if (f != NULL ) {
    f->allowed_seeks--;
    if (f->allowed_seeks <= 0 && file_to_compact_ == NULL) {
      file_to_compact_ = f;
      file_to_compact_level_ = stats.seek_file_level;
      return true ;
    }
  }
  return false ;
}
```
* 手动调用compact，compact指定的range


compact目标文件的选取策略
---------------
* imm_ != null，dump immutable memtable文件
* 执行外部调用的compact(如果指定的范围过大，则分批逐渐完成)
* 根据数据库状态选取：
  * level 1+文件大小总和超限，选取上次的compact_pointer_作为候选
  * level 0 文件总数超限，也根据compact_pointer_选取，同时选取level0上所有重叠的sst文件
  * 选取由于seek超限引起的compact。

```C++
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != NULL);
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0 );
    assert(level+ 1 < config::kNumLevels);
    c = new Compaction(level);

    // Pick the first file that comes after compact_pointer_[level]
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0 ) {
        c->inputs_[ 0 ].push_back(f);
        break ;
      }
    }
    if (c->inputs_[0 ].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[ 0 ].push_back(current_->files_[level][ 0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(level);
    c->inputs_[ 0 ].push_back(current_->file_to_compact_);
  } else {
    return NULL ;
  }

  c->input_version_ = current_;
  c->input_version_->Ref();

  // Files in level 0 may overlap each other, so pick up all overlapping ones
  if (level == 0 ) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[ 0 ], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    current_->GetOverlappingInputs( 0 , &smallest, &largest, &c->inputs_[0 ]);
    assert(!c->inputs_[ 0 ].empty());
  }

  SetupOtherInputs(c);

  return c;
}
```

```C++
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;
  GetRange(c->inputs_[ 0], &smallest, &largest);

  current_->GetOverlappingInputs(level+ 1, &smallest, &largest, &c->inputs_[1 ]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
  GetRange2(c->inputs_[ 0], c->inputs_[1 ], &all_start, &all_limit);

  // See if we can grow the number of inputs in "level" without
  // changing the number of "level+1" files we pick up.
  if (!c->inputs_[ 1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[ 0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[ 1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size < kExpandedCompactionByteSizeLimit) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level+ 1, &new_start, &new_limit,
                                     &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+ %ld bytes) to %d +%d ( %ld+%ld bytes)\n",
            level,
            int(c->inputs_[0 ].size()),
            int(c->inputs_[1 ].size()),
            long(inputs0_size), long (inputs1_size),
            int(expanded0.size()),
            int(expanded1.size()),
            long(expanded0_size), long (inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[ 0] = expanded0;
        c->inputs_[ 1] = expanded1;
        GetRange2(c->inputs_[ 0], c->inputs_[1 ], &all_start, &all_limit);
      }
    }
  }

  // Compute the set of grandparent files that overlap this compaction
  // (parent == level+1; grandparent == level+2)
  if (level + 2 < config::kNumLevels) {
    current_->GetOverlappingInputs(level + 2, &all_start, &all_limit,
                                   &c->grandparents_);
  }

  if ( false) {
    Log(options_->info_log, "Compacting %d ' %s' .. '%s '",
        level,
        smallest.DebugString().c_str(),
        largest.DebugString().c_str());
  }

  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
}
```

compact memtable的过程
---------------

```C++
Status DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != NULL );

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  Status s = WriteLevel0Table(imm_, &edit, base);
  base->Unref();

  if (s.ok() && shutting_down_.Acquire_Load()) {
    s = Status::IOError( "Deleting DB during memtable compaction" );
  }

  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber( 0 );
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {
    // Commit to the new state
    imm_->Unref();
    imm_ = NULL ;
    has_imm_.Release_Store( NULL );
    DeleteObsoleteFiles();
  }

  return s;
}

Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table # %llu: started" ,
      ( unsigned long long) meta.number);

  Status s;
  {
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table # %llu: %lld bytes %s " ,
      ( unsigned long long) meta.number,
      ( unsigned long long) meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);


  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0 ;
  if (s.ok() && meta.file_size > 0 ) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != NULL ) {
      // 如果没有重合，尽可能的放到更高的level(level <=2，因为过高不利于清除无效记录
      // 占用大量空间且不利于读，参见IsBaseLevelForKey实现)
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```

```C++
// Maximum level to which a new compacted memtable is pushed if it
// does not create overlap.  We try to push to level 2 to avoid the
// relatively expensive level 0=>1 compactions and to avoid some
// expensive manifest file operations.  We do not push all the way to
// the largest level since that can generate a lot of wasted disk
// space if the same key space is being repeatedly overwritten.
static const int kMaxMemCompactLevel = 2 ;
```

```C++
int Version::PickLevelForMemTableOutput(
    const Slice& smallest_user_key,
    const Slice& largest_user_key) {
  int level = 0 ;
  if (!OverlapInLevel( 0 , &smallest_user_key, &largest_user_key)) {
    // Push to next level if there is no overlap in next level,
    // and the #bytes overlapping in the level after that are limited.
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0 , static_cast <ValueType>( 0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {
      if (OverlapInLevel(level + 1 , &smallest_user_key, &largest_user_key)) {
        break ;
      }
      GetOverlappingInputs(level + 2 , &start, &limit, &overlaps);
      const int64_t sum = TotalFileSize(overlaps);
      if (sum > kMaxGrandParentOverlapBytes) {
        break ;
      }
      level++;
    }
  }
  return level;
}
```


compact sstable过程
---------------

```C++
Status DBImpl::DoCompactionWork(CompactionState* compact) {
  const uint64_t start_micros = env_->NowMicros();
  int64_t imm_micros = 0;  // Micros spent doing imm_ compactions

  Log(options_.info_log,  "Compacting %d@ %d + %d @%d files" ,
      compact->compaction->num_input_files( 0),
      compact->compaction->level(),
      compact->compaction->num_input_files( 1),
      compact->compaction->level() + 1);

  assert(versions_->NumLevelFiles(compact->compaction->level()) > 0);
  assert(compact->builder == NULL);
  assert(compact->outfile == NULL);
  if (snapshots_.empty()) {
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    compact->smallest_snapshot = snapshots_.oldest()->number_;
  }

  // Release mutex while we're actually doing the compaction work
  mutex_.Unlock();

  // 逻辑上是一个(key单调递增, sequence number单调递减)序列
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  input->SeekToFirst();
  Status status;
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  for (; input->Valid() && !shutting_down_.Acquire_Load(); ) {
    // Prioritize immutable compaction work
    if (has_imm_.NoBarrier_Load() != NULL ) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != NULL ) {
        CompactMemTable();
        bg_cv_.SignalAll();  // Wakeup MakeRoomForWrite() if necessary
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    Slice key = input->key();
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != NULL) {
      // 避免grandparent层有过多的重叠，提前关闭此文件
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }
 
    // Handle key/value, add to state, etc.
    bool drop = false ;
    if (!ParseInternalKey(key, &ikey)) {
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key,
                                     Slice(current_user_key)) != 0) {
        // First occurrence of this user key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }

      if (last_sequence_for_key <= compact->smallest_snapshot) {
        // Hidden by an newer entry for same user key
        // 上次的key相同，且其序列号小于所有snapshot最小序列号，可丢弃
        drop = true;    // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        // 第三个条件的含义是如果更高level可能有重复值，先不删除，否则会引起逻辑错误(不可见的变成可见)
        // IsBaseLevelForKey为保守查找，只查找文件的范围，如果包含则认为可能有重复
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;
    }

    if (!drop) {
      // Open output file if necessary
      if (compact->builder == NULL ) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());

      // Close output file if it is big enough
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }

    input->Next();
  }

  if (status.ok() && shutting_down_.Acquire_Load()) {
    status = Status::IOError( "Deleting DB during compaction" );
  }
  if (status.ok() && compact->builder != NULL) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  delete input;
  input = NULL;

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros - imm_micros;
  for ( int which = 0 ; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      stats.bytes_read += compact->compaction->input(which, i)->file_size;
    }
  }
  for ( size_t i = 0 ; i < compact->outputs.size(); i++) {
    stats.bytes_written += compact->outputs[i].file_size;
  }

  mutex_.Lock();
  stats_[compact->compaction->level() + 1].Add(stats);

  if (status.ok()) {
    status = InstallCompactionResults(compact);
  }
  VersionSet::LevelSummaryStorage tmp;
  Log(options_.info_log,
      "compacted to: %s" , versions_->LevelSummary(&tmp));
  return status;
}
```

compact的遍历通过iterator实现：
* level 0每个memtable文件作为单独的TowLevelIterator，因为key可能有重合
* 其他每个level所有文件作为一个TowLevelIterator，因为key没有重合
* 将所有level的iterator整合成MergingIterator

```C++
Iterator* VersionSet::MakeInputIterator(Compaction* c) {
  ReadOptions options;
  options.verify_checksums = options_->paranoid_checks;
  options.fill_cache = false;

  // Level-0 files have to be merged together.  For other levels,
  // we will make a concatenating iterator per level.
  // TODO (opt): use concatenating iterator for level-0 if there is no overlap
  const int space = (c->level() == 0 ? c->inputs_[0].size() + 1 : 2 );
  Iterator** list = new Iterator*[space];
  int num = 0;
  for ( int which = 0 ; which < 2; which++) {
    if (!c->inputs_[which].empty()) {
      if (c->level() + which == 0 ) {
        const std::vector<FileMetaData*>& files = c->inputs_[which];
        for (size_t i = 0; i < files.size(); i++) {
          list[num++] = table_cache_->NewIterator(
              options, files[i]->number, files[i]->file_size);
        }
      } else {
        // Create concatenating iterator for the files from this level
        list[num++] = NewTwoLevelIterator(
            new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
            &GetFileIterator, table_cache_, options);
      }
    }
  }
  assert(num <= space);
  Iterator* result = NewMergingIterator(&icmp_, list, num);
  delete[] list;
  return result;
}
```

compact过程中，生成的新文件在达到size上限时，同时检查是否与其下一层有过多的重合，如果达到上限(等于级数，10倍)，提前结束当前文件，开始新的文件，这样避免下次compact有过多的IO。
```C++
 // Maximum bytes of overlaps in grandparent (i.e., level+2) before we
// stop building a single file in a level->level+1 compaction.
static const int64_t kMaxGrandParentOverlapBytes = 10 * kTargetFileSize;

bool Compaction::ShouldStopBefore(const Slice& internal_key) {
  // Scan to find earliest grandparent file that contains key.
  const InternalKeyComparator* icmp = &input_version_->vset_->icmp_;
  while (grandparent_index_ < grandparents_.size() &&
      icmp->Compare(internal_key,
                    grandparents_[grandparent_index_]->largest.Encode()) > 0) {
    if (seen_key_) {
      overlapped_bytes_ += grandparents_[grandparent_index_]->file_size;
    }
    grandparent_index_++;
  }
  seen_key_ = true;

  if (overlapped_bytes_ > kMaxGrandParentOverlapBytes) {
    // Too much overlap for current output; start new output
    overlapped_bytes_ = 0;
    return true ;
  } else {
    return false ;
  }
}
```

IsBaseLevelForKey为检查更高level中是否可能存在相同的key，判断条件为判断每层中每个文件的key起始范围是否包含此key，没有检查bloom filter。疑问：是否此函数用处不大，大多数情况都是返回false？
```C++
Compaction::Compaction( int level)
    : level_(level),
      max_output_file_size_(MaxFileSizeForLevel(level)),
      input_version_( NULL),
      grandparent_index_( 0),
      seen_key_( false),
      overlapped_bytes_( 0) {
  for ( int i = 0 ; i < config::kNumLevels; i++) {
    level_ptrs_[i] = 0; // 一次compact初始化一次
  }
}

bool Compaction::IsBaseLevelForKey(const Slice& user_key) {
  // Maybe use binary search to find right entry instead of linear search?
  const Comparator* user_cmp = input_version_->vset_->icmp_.user_comparator();
  for ( int lvl = level_ + 2 ; lvl < config::kNumLevels; lvl++) {
    const std::vector<FileMetaData*>& files = input_version_->files_[lvl];
    for (; level_ptrs_[lvl] < files.size(); ) {
      FileMetaData* f = files[level_ptrs_[lvl]];
      if (user_cmp->Compare(user_key, f->largest.user_key()) <= 0) {
        // We've advanced far enough
        if (user_cmp->Compare(user_key, f->smallest.user_key()) >= 0) {
          // Key falls in this file's range, so definitely not base level
          return false ; // 为何不使用bloom filter？可能是因为compaction并无随机IO需要避免，计算hash的开销得不偿失？
        }
        break;
      }
      level_ptrs_[lvl]++; // 由于compact的iterator的key是递增的，此处记住每层上次查询的位置，从而每层总共只需要一次遍历
    }
  }
  return true;
}
```
