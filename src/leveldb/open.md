# 主要流程

Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr)，是LevelDB的入口函数，所有的操作之前必须open，在open内部有一系列的恢复、压缩等重要工作，更是LevelDB学习的重要入口。

Open内部的主要流程如下：

1. DBImpl* impl = new DBImpl(options, dbname)，DBImpl继承自DB，包含读写等操作的全部实现。
2. impl->Recover(&edit)，如果DB是重启会根据manifest等db的元数据信息恢复DB。
   1. versions_->Recover()，恢复manifest记录的sst文件、Version、VersionSet等落盘数据和元信息。
   2. RecoverLogFile(logs[i], edit, &max_sequence)，恢复binlog、memtable等数据。
3. 成员变量更新到最新值、删除多余文件、调度compatch线程（根据标准是否开启）等收尾工作。

# 源码解析

本部分主要是追踪源码，看一下open内部的实现细节（受篇幅限制，仅展示重要的源码片段）。

##### DBImpl 初始化

先看一下DBImpl包含哪些成员变量，方便后续内容理解。

```c++
DBImpl::DBImpl(const Options& raw_options, const std::string& dbname)
    : env_(raw_options.env),					// 系统环境封装，一般使用默认值即可
      internal_comparator_(raw_options.comparator),		// 比较器，默认字典序
      internal_filter_policy_(raw_options.filter_policy),	// 默认boolm过滤器
      options_(SanitizeOptions(dbname, &internal_comparator_,	
                &internal_filter_policy_, raw_options)),
      owns_info_log_(options_.info_log != raw_options.info_log),	// 运行之日，默认LOG，在SanitizeOptions初始化时，会将上次的LOG rename为LOG.old
      owns_cache_(options_.block_cache != raw_options.block_cache),	// 
      dbname_(dbname),							// db name
      db_lock_(NULL),							// 文件锁
      shutting_down_(NULL),						// shut down标志
      bg_cv_(&mutex_),							// 压缩线程与主线程之间同步的信号量
      mem_(new MemTable(internal_comparator_)),				// memtable
      imm_(NULL),							// immutable
      logfile_(NULL),							// binlog
      logfile_number_(0),						// binlog file number，递增
      log_(NULL),							// log::Writer写binlog用的
      seed_(0),								// 迭代器使用(后续介绍)
      tmp_batch_(new WriteBatch),					// write batch
      bg_compaction_scheduled_(false),					// 是否有compact金正
      manual_compaction_(NULL) {					// compact_range
  mem_->Ref();
  has_imm_.Release_Store(NULL);

  // Reserve ten files or so for other uses and give the rest to TableCache.
  // 每一个文件有一个cache
  const int table_cache_size = options_.max_open_files - kNumNonTableCacheFiles;
  table_cache_ = new TableCache(dbname_, &options_, table_cache_size);

  // VersionSet 记录了DB的变更记录
  versions_ = new VersionSet(dbname_, &options_, table_cache_,
                             &internal_comparator_);
}
```



##### DBImpl::Recover(VersionEdit* edit)

```c++
Status DBImpl::Recover(VersionEdit* edit) {
  /* 略 */
  
  // 通过manifest 文件中的versionEdit恢复
  s = versions_->Recover();
  
  // 从binlog恢复
  if (s.ok()) {
    SequenceNumber max_sequence(0);
    
    // PrevLogNumber 兼容老版本
    const uint64_t min_log = versions_->LogNumber();
    const uint64_t prev_log = versions_->PrevLogNumber();
    std::vector<std::string> filenames;
    s = env_->GetChildren(dbname_, &filenames);
    /*略*/
    std::set<uint64_t> expected;
    
    // 遍历VersionSet中所有的version，把所有在用的文件写入expected
    versions_->AddLiveFiles(&expected);
    uint64_t number;
    FileType type;
    std::vector<uint64_t> logs;
    
    // 遍历DB的所有文件
    for (size_t i = 0; i < filenames.size(); i++) {
      if (ParseFileName(filenames[i], &number, &type)) {
        expected.erase(number);
        
        // 将binlog文件取出来，文件号（number）>= min_log或者 == prev_log 表示已经恢复过
        if (type == kLogFile && ((number >= min_log) || (number == prev_log)))
          logs.push_back(number);
      }
    }
    
    // manifest记录的文件在目录中没有找到，说明DB有文件丢失
    if (!expected.empty()) {
      char buf[50];
      snprintf(buf, sizeof(buf), "%d missing files; e.g.",
               static_cast<int>(expected.size()));
      return Status::Corruption(buf, TableFileName(dbname_, *(expected.begin())));
    }

    // 处理已经落盘的binlog，但是没记录在manifest中，
    std::sort(logs.begin(), logs.end());
    for (size_t i = 0; i < logs.size(); i++) {
      s = RecoverLogFile(logs[i], edit, &max_sequence);

      // 跳过一个文件号
      versions_->MarkFileNumberUsed(logs[i]);
    }

		// 更新seq为未处理的binlog的最大seq
    if (s.ok()) {
      if (versions_->LastSequence() < max_sequence) {
        versions_->SetLastSequence(max_sequence);
      }
    }
  }

  return s;
}
```

接下来重点看一下versions_->Recover() 和 RecoverLogFile是如何从manifest 和 binlog来恢复DB的。

###### VersionSet::Recover()

```c++
Status VersionSet::Recover() {
  /* versionEdit在version上操作的一个工具类
   * 主要工作有两个：
   * 1. Apply(VersionEdit* edit)：将edit操作在this, current_上，添加、删除文件。
   * 2. SaveTo(Version* v)：将current_的当前状态dump到*v中
   */
  Builder builder(this, current_);
  {
    LogReporter reporter;
    reporter.status = &s;
    log::Reader reader(file, &reporter, true/*checksum*/, 0/*initial_offset*/);
    Slice record;
    std::string scratch;
    // 读取manifest文件的versionEdit记录
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {
      VersionEdit edit;
      s = edit.DecodeFrom(record);
      if (s.ok()) {
        if (edit.has_comparator_ &&
            edit.comparator_ != icmp_.user_comparator()->Name()) {
          s = Status::InvalidArgument(
              edit.comparator_ + " does not match existing comparator ",
              icmp_.user_comparator()->Name());
        }
      }

      // 将edit应用在current_（Builder初始化参数）上
      if (s.ok()) {
        builder.Apply(&edit);
      }

      // 记录中间状态，最后更新dn
      if (edit.has_log_number_) {
        log_number = edit.log_number_;
        have_log_number = true;
      }

      if (edit.has_prev_log_number_) {
        prev_log_number = edit.prev_log_number_;
        have_prev_log_number = true;
      }

      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;
        have_next_file = true;
      }

      if (edit.has_last_sequence_) {
        last_sequence = edit.last_sequence_;
        have_last_sequence = true;
      }
    }
  }
  delete file;
  file = NULL;

  if (s.ok()) {
    // 下面的变量是必须有的字段，否则元信息有损坏
    if (!have_next_file) {
      s = Status::Corruption("no meta-nextfile entry in descriptor");
    } else if (!have_log_number) {
      s = Status::Corruption("no meta-lognumber entry in descriptor");
    } else if (!have_last_sequence) {
      s = Status::Corruption("no last-sequence-number entry in descriptor");
    }

    // 兼容旧版本
    if (!have_prev_log_number) {
      prev_log_number = 0;
    }
		
    // 递增一个文件号
    MarkFileNumberUsed(prev_log_number);
    MarkFileNumberUsed(log_number);
  }

  if (s.ok()) {
    Version* v = new Version(this);
    // 将最终的version，dump出来
    builder.SaveTo(v);
    /* 更新compaction_level_ ,compaction_score_ 
     * 表示当前DB的状态中最应该进行compact的level和层
     * level == 0，files_[level].size() / kL0_CompactionTrigger (默认4)
     * level >= 1, TotalFileSize(v->files_[level]) / MaxBytesForLevel(level)（10M/100M,7层，最大10T）
     */
    Finalize(v);
    /* 将v加入VersionSet（DBImpl的成员变量version_）的链表头部
     */
    AppendVersion(v);
    // 更新到最新值（manifest维护状态的最新值），如果binlog不用恢复，这些信息就是当前DB的
    manifest_file_number_ = next_file;
    next_file_number_ = next_file + 1;
    last_sequence_ = last_sequence;
    log_number_ = log_number;
    prev_log_number_ = prev_log_number;
  }

  return s;
}
```

###### DBImpl::RecoverLogFile(uint64_t log_number, VersionEdit* edit, SequenceNumber* max_sequence）

```c++
Status DBImpl::RecoverLogFile(uint64_t log_number,
                              VersionEdit* edit,
                              SequenceNumber* max_sequence) {
  /* 略 */
  // Read all the records and add to a memtable
  std::string scratch;
  Slice record;
  WriteBatch batch;
  
  // binlog 的内容一定是内存的数据，否则都会刷如磁盘了
  // 有可能在memtable，有可能在immutable中
  MemTable* mem = NULL;
  while (reader.ReadRecord(&record, &scratch) &&
         status.ok()) {
    // 每条记录保存前12个字节，8-byte sequence number followed by a 4-byte count
    if (record.size() < 12) {
      reporter.Corruption(
          record.size(), Status::Corruption("log record too small"));
      continue;
    }
    WriteBatchInternal::SetContents(&batch, record);

    if (mem == NULL) {
      mem = new MemTable(internal_comparator_);
      mem->Ref();
    }
    
    // 逐条写入memtable
    status = WriteBatchInternal::InsertInto(&batch, mem);
    MaybeIgnoreError(&status);
    if (!status.ok()) {
      break;
    }
    const SequenceNumber last_seq =
        WriteBatchInternal::Sequence(&batch) +
        WriteBatchInternal::Count(&batch) - 1;
    
    // 更新last_seq
    if (last_seq > *max_sequence) {
      *max_sequence = last_seq;
    }

    // 此时binlog没有读完，但是memtable已经写满
    // compact level 0，刷盘（后续介绍WriteLevel0Table）
    if (mem->ApproximateMemoryUsage() > options_.write_buffer_size) {
      status = WriteLevel0Table(mem, edit, NULL);
      if (!status.ok()) {
        break;
      }
      mem->Unref();
      mem = NULL;
    }
  }

  // 注意：重启之后将内存的数据写入磁盘
  if (status.ok() && mem != NULL) {
    status = WriteLevel0Table(mem, edit, NULL);
  }

  if (mem != NULL) mem->Unref();
  delete file;
  return status;
}
```

##### DB::Open

最后看一下open的整体流程：

```c++
Status DB::Open(const Options& options, const std::string& dbname,
                DB** dbptr) {
  *dbptr = NULL;

  DBImpl* impl = new DBImpl(options, dbname);
  impl->mutex_.Lock();
  VersionEdit edit;
  
  // recover manifest 和 binlog
  Status s = impl->Recover(&edit); // Handles create_if_missing, error_if_exists
  if (s.ok()) {
  	
    // 生成新的binlog文件，在这里可以看到，binlog 和 sst文件公用一个序列的文件号
    uint64_t new_log_number = impl->versions_->NewFileNumber();
    WritableFile* lfile;
    s = options.env->NewWritableFile(LogFileName(dbname, new_log_number),
                                     &lfile);
    if (s.ok()) {
      // 更新最新的信息
      edit.SetLogNumber(new_log_number);
      impl->logfile_ = lfile;
      impl->logfile_number_ = new_log_number;
      impl->log_ = new log::Writer(lfile);
      s = impl->versions_->LogAndApply(&edit, &impl->mutex_);
    }
    if (s.ok()) {
      /* 在DB目录下扫描所有文件，对于没有的文件删除 */
      impl->DeleteObsoleteFiles();
      /* 调度compact线程，根据标准决定是否执行压缩，后续介绍 */
      impl->MaybeScheduleCompaction();
    }
  }
  impl->mutex_.Unlock();
  if (s.ok()) {
    *dbptr = impl;
  } else {
    delete impl;
  }
  return s;
}
```

