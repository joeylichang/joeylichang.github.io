# Master Spilt Tablet 流程

Master侧拆分Tablet的流程，如下：

1. Tablet 状态必须是ready状态。
2. 向节点发送ComputeSplitKey请求进行拆分。
3. 拆分成功之后，unload_tablet。
4. 校验unload_tablet是否成功，检验方式是看存储目录是否有log文件，有的话表示拆分未完成。
   1. 校验失败，进行回滚，加载之前的tablet。
   2. 校验成功，继续。
5. 更新元数据添加联调tablet元数据。
6. 更新master内存数据。
7. 通过load_tablet流程加载两个新的tablet。
   1. 重点在这，

### kPreSplitTablet

```c++
void SplitTabletProcedure::PreSplitTabletPhaseHandler(const SplitTabletPhase&) {
  // 如果指定了split key，进行校验，从unload_tablet开始执行
  if (!split_key_.empty()) {
    if ((!tablet_->GetKeyStart().empty() && split_key_ <= tablet_->GetKeyStart()) ||
        (!tablet_->GetKeyEnd().empty() && split_key_ >= tablet_->GetKeyEnd())) {
      PROC_LOG(WARNING) << "invalid split key: " << split_key_ << ", tablet: " << tablet_;
      SetNextPhase(SplitTabletPhase::kEofPhase);
      return;
    }
    SetNextPhase(SplitTabletPhase::kUnLoadTablet);
  } else if (dispatch_split_key_request_) {
    // waiting RPC response
    return;
  } else {
    dispatch_split_key_request_ = true;
    
    // 向ts发送spilt的请求
    ComputeSplitKeyAsync();
  }
}
```



### kUnLoadTablet

```C++
void SplitTabletProcedure::UnloadTabletPhaseHandler(const SplitTabletPhase&) {
  if (!unload_proc_) {
    unload_proc_.reset(new UnloadTabletProcedure(tablet_, thread_pool_, true));
    PROC_LOG(INFO) << "Generate UnloadTablet SubProcedure: " << unload_proc_->ProcId();
    MasterEnv().GetExecutor()->AddProcedure(unload_proc_);
  }
  if (!unload_proc_->Done()) {
    return;
  }
  TabletNodePtr node = tablet_->GetTabletNode();
  if (tablet_->GetStatus() == TabletMeta::kTabletOffline) {
    SetNextPhase(SplitTabletPhase::kPostUnLoadTablet);
  } else {
    SetNextPhase(SplitTabletPhase::kEofPhase);
  }
}
```



### kPostUnLoadTablet

```c++
void SplitTabletProcedure::PostUnloadTabletPhaseHandler(const SplitTabletPhase&) {
  if (!TabletStatusCheck()) {
    SetNextPhase(SplitTabletPhase::kFaultRecover);
  } else {
    SetNextPhase(SplitTabletPhase::kUpdateMeta);
  }
}

bool SplitTabletProcedure::TabletStatusCheck() {
  leveldb::Env* env = io::LeveldbBaseEnv();
  std::vector<std::string> children;
  std::string tablet_path = FLAGS_tera_tabletnode_path_prefix + "/" + tablet_->GetPath();
  leveldb::Status status = env->GetChildren(tablet_path, &children);
  if (!status.ok()) {
    PROC_LOG(WARNING) << "[split] abort, " << tablet_
                      << ", tablet status check error: " << status.ToString();
    return false;
  }
  for (size_t i = 0; i < children.size(); ++i) {
    leveldb::FileType type = leveldb::kUnknown;
    uint64_t number = 0;
    
    // 检出存储目录下文件的类型，判断spilt是否成功
    if (ParseFileName(children[i], &number, &type) && type == leveldb::kLogFile) {
      PROC_LOG(WARNING) << "[split] abort, " << tablet_ << ", tablet log not clear.";
      return false;
    }
  }
  return true;
}
```



### kUpdateMeta

```c++
void SplitTabletProcedure::UpdateMeta() {
  std::vector<MetaWriteRecord> records;

  std::string parent_path = tablet_->GetPath();
  TablePtr table = tablet_->GetTable();
  std::string child_key_start = tablet_->GetKeyStart();
  std::string child_key_end = split_key_;
  for (int i = 0; i < 2; ++i) {
    TabletMeta child_meta;
    tablet_->ToMeta(&child_meta);
    child_meta.clear_parent_tablets();
    child_meta.set_status(TabletMeta::kTabletOffline);
    child_meta.add_parent_tablets(leveldb::GetTabletNumFromPath(parent_path));
    child_meta.set_path(leveldb::GetChildTabletPath(parent_path, table->GetNextTabletNo()));
    child_meta.mutable_key_range()->set_key_start(child_key_start);
    child_meta.mutable_key_range()->set_key_end(child_key_end);
    child_meta.set_size(tablet_->GetDataSize() / 2);
    child_meta.set_version(tablet_->Version() + 1);
    child_tablets_[i].reset(new Tablet(child_meta, table));
    child_key_start = child_key_end;
    child_key_end = tablet_->GetKeyEnd();
    PackMetaWriteRecords(child_tablets_[i], false, records);
  }

  UpdateMetaClosure done = std::bind(&SplitTabletProcedure::UpdateMetaDone, this, _1);
  PROC_LOG(INFO) << "[split] update meta async: " << tablet_;
  MasterEnv().BatchWriteMetaTableAsync(records, done, -1);
}

void SplitTabletProcedure::UpdateMetaDone(bool) {
  TabletMeta first_meta, second_meta;
  child_tablets_[0]->ToMeta(&first_meta);
  first_meta.set_status(TabletMeta::kTabletOffline);
  child_tablets_[1]->ToMeta(&second_meta);
  second_meta.set_status(TabletMeta::kTabletOffline);
  TablePtr table = tablet_->GetTable();
  child_tablets_[0]->LockTransition();
  child_tablets_[1]->LockTransition();

  tablet_->DoStateTransition(TabletEvent::kFinishSplitTablet);
  
  // 更新内存数据，去掉源tablet，增加两个新的tablet
  table->SplitTablet(tablet_, first_meta, second_meta, &child_tablets_[0], &child_tablets_[1]);
  PROC_LOG(INFO) << "split finish, " << tablet_ << ", try load child tablet,"
                 << "\nfirst: " << child_tablets_[0] << "\nsecond: " << child_tablets_[1];
  SetNextPhase(SplitTabletPhase::kLoadTablets);
}
```



### kLoadTablets

```c++
void SplitTabletProcedure::LoadTabletsPhaseHandler(const SplitTabletPhase&) {
  // 加载两个新的tablet
  if (!load_procs_[0] && !load_procs_[1]) {
    TabletNodePtr node = tablet_->GetTabletNode();
    // try load tablet at the origin tabletnode considering cache locality
    load_procs_[0].reset(new LoadTabletProcedure(child_tablets_[0], node, thread_pool_ /*, true*/));
    load_procs_[1].reset(new LoadTabletProcedure(child_tablets_[1], node, thread_pool_ /*, true*/));
    PROC_LOG(INFO) << "Generate LoadTablet SubProcedure1: " << load_procs_[0]->ProcId();
    PROC_LOG(INFO) << "Generate LoadTablet SubProcedure2, " << load_procs_[1]->ProcId();
    MasterEnv().GetExecutor()->AddProcedure(load_procs_[0]);
    MasterEnv().GetExecutor()->AddProcedure(load_procs_[1]);
  }
  PROC_CHECK(load_procs_[0] && load_procs_[1]);
  SetNextPhase(SplitTabletPhase::kEofPhase);
}
```

