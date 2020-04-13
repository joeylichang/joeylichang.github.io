# Master Create Table 流程

Master侧对 Create Table分为了4个阶段，既kPrepare, kUpdateMeta, kLoadTablets, kEofPhase，并对四个节点设置了对应的回调，内部通过vector模拟了一个队列，按顺序push四种状态，pop时执行对应的回调。RunNextStage负责状态pop，以及回调的执行。

### PreCheckHandler

kPrepare阶段，Master对请求参数进行校验，元数据组织等工作。

```C++
void CreateTableProcedure::PreCheckHandler(const CreateTablePhase&) {
  {
    TablePtr table;
    // 查重
    if (MasterEnv().GetTabletManager()->FindTable(request_->table_name(), &table)) {
      EnterEofPhaseWithResponseStatus(kTableExist);
      return;
    }
    // 权限校验
    if (FLAGS_tera_acl_enabled && !MasterEnv().GetMaster()->IsRootUser(request_->user_token()) &&
        FLAGS_tera_only_root_create_table) {
      EnterEofPhaseWithResponseStatus(kNotPermission);
      return;
    }
  }

  // 如果之前有同名的表信息，则删除
  if (!io::MoveEnvDirToTrash(request_->table_name())) {
    PROC_LOG(ERROR) << "Fail to create table: " << request_->table_name()
                    << ", cannot move old table dir to trash";
    EnterEofPhaseWithResponseStatus(kTableExist);
    return;
  }

  // 要创建的tablet数据，参数中指定
  int32_t tablet_num = request_->delimiters_size() + 1;
  bool delivalid = true;
  for (int32_t i = 1; i < tablet_num - 1; i++) {
    // TODO: Use user defined comparator
    if (request_->delimiters(i) <= request_->delimiters(i - 1)) {
      delivalid = false;
      break;
    }
  }
  
  // 默认初始最多创建10w 个 tablet
  if (tablet_num > FLAGS_tera_max_pre_assign_tablet_num || !delivalid ||
      request_->schema().locality_groups_size() < 1) {
    if (tablet_num > FLAGS_tera_max_pre_assign_tablet_num) {
      PROC_LOG(WARNING) << "Too many pre-create tablets " << tablet_num;
    } else if (!delivalid) {
      PROC_LOG(WARNING) << "Invalid delimiters for " << request_->table_name();
    } else {
      PROC_LOG(WARNING) << "No LocalityGroupSchema for " << request_->table_name();
    }
    EnterEofPhaseWithResponseStatus(kInvalidArgument);
    return;
  }

  const std::string& table_name = request_->table_name();
  StatusCode status = kMasterOk;
  tablets_.reserve(tablet_num);
  meta_records_.reserve(tablet_num + 1);

  // 内存中创建表
  table_ = TabletManager::CreateTable(table_name, request_->schema(), kTableEnable);
  table_->LockTransition();
  // 内存中创建tablet，并且将table 和 tablet的原信息组织好，准备下阶段去更新元数据表
  PackMetaWriteRecords(table_, false, meta_records_);
  for (int32_t i = 1; i <= tablet_num; ++i) {
    std::string path = leveldb::GetTabletPathFromNum(request_->table_name(), i);
    const std::string& start_key = (i == 1) ? "" : request_->delimiters(i - 2);
    const std::string& end_key = (i == tablet_num) ? "" : request_->delimiters(i - 1);
    TabletMeta meta;
    TabletManager::PackTabletMeta(&meta, table_name, start_key, end_key, path, "",
                                  TabletMeta::kTabletOffline,
                                  FLAGS_tera_tablet_write_block_size * 1024);
    TabletPtr tablet = table_->AddTablet(meta, &status);
    if (!tablet) {
      PROC_LOG(WARNING) << "add tablet failed" << meta.path()
                        << ", errcode: " << StatusCodeToString(status);
      EnterEofPhaseWithResponseStatus(status);
      MasterEnv().GetTabletManager()->DeleteTable(table_name_, &status);
      return;
    }

    tablet->LockTransition();
    // 打包元数据
    PackMetaWriteRecords(tablet, false, meta_records_);
    tablets_.emplace_back(tablet);
  }
  // 内存中添加table到tablemanager
  if (!MasterEnv().GetTabletManager()->AddTable(table_, &status)) {
    PROC_LOG(ERROR) << "Fail to create table: " << request_->table_name()
                    << ", table already exist";
    EnterEofPhaseWithResponseStatus(kTableExist);
    return;
  }

  // 设置下个阶段的状态
  SetNextPhase(CreateTablePhase::kUpdateMeta);
}
```



### UpdateMetaHandler

kUpdateMeta阶段，将上阶段准备的选信息，去更新元表。

```c++
void CreateTableProcedure::UpdateMetaHandler(const CreateTablePhase&) {
  if (update_meta_) {
    return;
  }
  update_meta_.store(true);
  PROC_LOG(INFO) << "table: " << table_name_ << " begin to update meta";
  UpdateMetaClosure closure = std::bind(&CreateTableProcedure::UpdateMetaDone, this, _1);
  // 调用客户端，批量更新元数据到元表中
  MasterEnv().BatchWriteMetaTableAsync(meta_records_, closure, FLAGS_tera_master_meta_retry_times);
}

// 更新完成的回调函数
void CreateTableProcedure::UpdateMetaDone(bool succ) {
  if (!succ) {
    PROC_LOG(WARNING) << "fail to update meta";
    EnterEofPhaseWithResponseStatus(kMetaTabletError);
    return;
  }
  LOG(INFO) << "create table " << table_->GetTableName() << " update meta success";
  EnterPhaseAndResponseStatus(kMasterOk, CreateTablePhase::kLoadTablets);
}
```



### LoadTabletsHandler

kLoadTablets阶段，选择TabletNode进行加载tablet。

```c++
void CreateTableProcedure::LoadTabletsHandler(const CreateTablePhase&) {
  Scheduler* size_scheduler = MasterEnv().GetSizeScheduler().get();
  for (size_t i = 0; i < tablets_.size(); i++) {
    CHECK(tablets_[i]->GetStatus() == TabletMeta::kTabletOffline) << tablets_[i]->GetPath();
    TabletNodePtr dest_node;
    // 选择节点，考虑的节点的容量进行选择，在LB部分介绍选择节点的策略
    if (!MasterEnv().GetTabletNodeManager()->ScheduleTabletNodeOrWait(
            size_scheduler, table_name_, tablets_[i], false, &dest_node)) {
      LOG(ERROR) << "no available tabletnode, abort load tablet: " << tablets_[i];
      continue;
    }
    // 创建LoadTabletProcedure 去执行任务，后续介绍
    std::shared_ptr<LoadTabletProcedure> proc(
        new LoadTabletProcedure(tablets_[i], dest_node, thread_pool_));
    MasterEnv().GetExecutor()->AddProcedure(proc);
  }
  SetNextPhase(CreateTablePhase::kEofPhase);
}
```



### EofHandler

kEofPhase阶段，资源回收，和完成状态的设定。

注意：此时创建表结束了，但是tablet 未必都加载完成了。

```c++
void CreateTableProcedure::EofHandler(const CreateTablePhase&) {
  PROC_LOG(INFO) << "create table: " << table_name_ << " finish";
  done_.store(true);
  // unlike DisableTableProcedure, here we do not ensure that all tablets been
  // loaded successfully
  // and just finish the CreateTableProcedure early as all LoadTabletProcedure
  // been added to ProcedureExecutor.
  if (table_ && table_->InTransition()) {
    table_->UnlockTransition();
  }
  rpc_closure_->Run();
}
```

