# 触发条件

1. DB::Open之后，查看是否到达compact条件，达到标准后触发。
2. DB::Write之前，MakeRoomForWrite判断是否触发compact。
3. DB::Get之后，统计内存是否命中，如果没有且Seek文件超过阈值时触发。
4. DB::CompactRange，触发压缩线程。

# 压缩主要流程

1. Compact immutable。
2. PickCompaction：
   1. 文件大小、文件seek次数。
   2. level、level + 1放大一次。
   3. 考虑Level + 2 影响范围，否则后续compact任务太重。
3. DoCompactionWork
   1. 先check immutable，有的话刷磁盘。
   2. 迭代处理level、level+1，如果当前处理key 在 level + 2 上影响面过大也停止。
   3. apply edit，更改manifest。

# 源码分析

大部分的compact（除CompactRange之外）是从调用void DBImpl::MaybeScheduleCompaction() 开始，其内部调用env_->Schedule(&DBImpl::BGWork, this)，env_->Schedule内部通过锁、信号量、队列启动一个bg线程执行compact，且保证（队列）每次执行一个任务。compact的核心逻辑在void DBImpl::BackgroundCompaction()。

##### void DBImpl::BackgroundCompaction()

```c++
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  // 刷内存数据到level 0
  if (imm_ != NULL) {
    CompactMemTable();
    return;
  }

  Compaction* c;
  // is_manual CompactRange标志，不做重点介绍
  if (is_manual) {
    /* ... */
  } else {
    // 选取需要压缩的信息
    c = versions_->PickCompaction();
  }

  Status status;
  if (c == NULL) {
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {
    // level + 1 没有重叠的key，直接向下层平移 
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    c->edit()->DeleteFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size,
                       f->smallest, f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    /* 略 */
  } else {
    CompactionState* compact = new CompactionState(c);
    // compact 核心逻辑
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    // 清理过程文件在dbimpl的记录
    CleanupCompaction(compact);
    c->ReleaseInputs();
    // 实际删除无用的文件
    DeleteObsoleteFiles();
  }
  delete c;
	/* 略 */
}
```

下面重点看一下CompactMemTable() 和 PickCompaction()、 DoCompactionWork(compact)。

##### void DBImpl::CompactMemTable()

```c++
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != NULL);

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  
  // 刷内存数据到磁盘，文件等DB的元信息变更写入edit
  Status s = WriteLevel0Table(imm_, &edit, base);
  base->Unref();

  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    // edit 在versions_上应用，内部会更新manifest
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {
    // Commit to the new state
    imm_->Unref();
    imm_ = NULL;
    has_imm_.Release_Store(NULL);
    DeleteObsoleteFiles();
  } else {
    RecordBackgroundError(s);
  }
}
```

###### DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,Version* base)

```c++
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();			// 递增+1
  pending_outputs_.insert(meta.number);					// 记录正在用还没加入edit的文件
  Iterator* iter = mem->NewIterator();

  Status s;
  {
    mutex_.Unlock();
    // 创建文件并写入磁盘
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }
  delete iter;
  pending_outputs_.erase(meta.number);

  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != NULL) {
      /* 选择一个level加入文件，原则如下：
       * 1. 与 level 0 有交集，直接选level 0
       * 2. 与 level 0 没有交集，继续往下找，直到遇到有交集的上一层为止
       * 3. 最多往下透传2层且 与 grandparent层（往下两层）的交集数据大小 < kMaxGrandParentOverlapBytes(默认 25 * 32MB)， 防止compact任务太重
       */
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
  }
  
  return s;
}
```



##### Compaction* VersionSet::PickCompaction()

```c++
// 压缩过程的状态记录信息
class Compaction {
	int level_;
  uint64_t max_output_file_size_;
  Version* input_version_;
  VersionEdit edit_;

  // Each compaction reads inputs from "level_" and "level_+1"
  std::vector<FileMetaData*> inputs_[2];      // The two sets of inputs

  // State used to check for number of of overlapping grandparent files
  // (parent == level_ + 1, grandparent == level_ + 2)
  std::vector<FileMetaData*> grandparents_;
  size_t grandparent_index_;  // Index in grandparent_starts_
  bool seen_key_;             // Some output key has been seen
  int64_t overlapped_bytes_;  // Bytes of overlap between current output
                              // and grandparent files

  // level_ptrs_ holds indices into input_version_->levels_: our state
  // is that we are positioned at one of the file ranges for each
  // higher level than the ones involved in this compaction (i.e. for
  // all L >= level_ + 2).
  size_t level_ptrs_[config::kNumLevels];
}

Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  /*
   * 两种压缩触发标准
   * 1. level 的文件总大小 / 当前level（层数有关）默认的闸值，每次apply edit之后，调用Finalize 会计算每层的分数，记录下最大的分数 和 相关的level（compaction_level_）
   * 2. seek文件次数达到闸值（默认1 << 30），将文件记录在file_to_compact_
   */
  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != NULL);
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level+1 < config::kNumLevels);
    c = new Compaction(level);

    // 按层遍历所有文件
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      // compact_pointer_ 记录上次压缩的进度（key）
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {、
        // inputs_[0] 是level，inputs_[1] 是 level + 1
        c->inputs_[0].push_back(f);
        break;
      }
    }
    if (c->inputs_[0].empty()) {
      // 从第一个文件开始
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(level);
    // 直接用统计（每次读之后都会统计）的file_to_compact_ 作为inputs_[0]
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return NULL;
  }

  c->input_version_ = current_;
  c->input_version_->Ref();

  // level 0 比较特殊，因为可能有重叠
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }

  /* 在level + 1选取文件放入inputs_[1]，有一层优化对compact的放大（可能改变inputs_[0]），流程如下
   * 1. 通过inputs_[0]，获取key范围，再根据key范围获取inputs_[1]。
   * 2. 通过inputs_[0]、inputs_[1] 获取更大的key范围，再去level中重新拉去文件list expanded0。
   * 3. 如果expanded0 变大了（表示又有新文件加入）且 如果expanded0 + inputs_[1] < 闸值（默认25 * 32M，太大加重压缩任务），通过expanded0 去 level + 1 中获取通过expanded1。
   * 4. 如果expanded1 与 inputs_[1] 相同则，表示没有再放大，此时inputs_[0] = expanded0，inputs_[1] = expanded1。
   * grandparents_ 计算在level + 2 设计的文件
   * 更新compact_pointer_
   * 设置了本次compact的edit_的SetCompactPointer(level, largest)
   */
  SetupOtherInputs(c);

  return c;
}
```



##### DBImpl::DoCompactionWork(CompactionState* compact)

```c++
struct DBImpl::CompactionState {
  Compaction* const compaction;

  // Sequence numbers < smallest_snapshot are not significant since we
  // will never have to service a snapshot below smallest_snapshot.
  // Therefore if we have seen a sequence number S <= smallest_snapshot,
  // we can drop all entries for the same key with sequence numbers < S.
  SequenceNumber smallest_snapshot;

  // Files produced by compaction
  struct Output {
    uint64_t number;
    uint64_t file_size;
    InternalKey smallest, largest;
  };
  std::vector<Output> outputs;
  // State kept for output being generated
  WritableFile* outfile;
  TableBuilder* builder;

  uint64_t total_bytes;
  Output* current_output() { return &outputs[outputs.size()-1]; }

};


Status DBImpl::DoCompactionWork(CompactionState* compact) {
  assert(versions_->NumLevelFiles(compact->compaction->level()) > 0);
  assert(compact->builder == NULL);
  assert(compact->outfile == NULL);
  if (snapshots_.empty()) {
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    compact->smallest_snapshot = snapshots_.oldest()->number_;
  }

  
  mutex_.Unlock();
  // 多文件混合迭代器，后面介绍
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  input->SeekToFirst();
  Status status;
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  for (; input->Valid() && !shutting_down_.Acquire_Load(); ) {
    // 此时如果写的速度快的话，immutable已经满了，需要先刷一下
    if (has_imm_.NoBarrier_Load() != NULL) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != NULL) {
        CompactMemTable();
        bg_cv_.SignalAll();  // Wakeup MakeRoomForWrite() if necessary
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    // 如果对于level + 2 层影响过大（默认超过 20 * 32M）则停止本次compact
    Slice key = input->key();
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != NULL) {
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }

    // Handle key/value, add to state, etc.
    bool drop = false;
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
        // smallest_snapshot 最新的snapshot，小于他的seq都应该删除
        drop = true;    // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 // level + 2 往下层没有key
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        drop = true;
      }
      last_sequence_for_key = ikey.sequence;
    }

    if (!drop) {
      // Open output file if necessary
      if (compact->builder == NULL) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      // 落盘
      compact->builder->Add(key, input->value());

      // 单文件默认最大32M
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

  /* 略 */
  if (status.ok()) {
    // versionEdut apply
    status = InstallCompactionResults(compact);
  }
  return status;
}
```

