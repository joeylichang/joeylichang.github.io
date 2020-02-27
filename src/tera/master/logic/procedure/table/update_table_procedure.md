

# Master Update Table 流程

Master侧update table分为下面四个阶段进行：

1. kPrepare：信息校验。
2. kUpdateMeta：更新table的元数据。
3. kTabletsSchemaSyncing：同步给tablet。
4. kEofPhase：设置结束标志。

### kPrepare:PrepareHandler

```c++
void UpdateTableProcedure::PrepareHandler(const UpdateTablePhase&) {
  if (!MasterEnv().GetMaster()->HasPermission(request_, table_, "update table")) {
    EnterPhaseWithResponseStatus(kNotPermission, UpdateTablePhase::kEofPhase);
    return;
  }

  if (request_->schema().locality_groups_size() < 1) {
    PROC_LOG(WARNING) << "No LocalityGroupSchema for " << request_->table_name();
    EnterPhaseWithResponseStatus(kInvalidArgument, UpdateTablePhase::kEofPhase);
    return;
  }
  
  // 双buffer，保存两份schema，防止回滚
  if (!table_->PrepareUpdate(request_->schema())) {
    // another schema-update is doing...
    PROC_LOG(INFO) << "[update] no concurrent schema-update, table:" << table_;
    EnterPhaseWithResponseStatus(kTableNotSupport, UpdateTablePhase::kEofPhase);
    return;
  }
  
  // IsUpdateCf校验两个schema的列是否不同，如果相同表示没有更新
  if (FLAGS_tera_online_schema_update_enabled && table_->GetStatus() == kTableEnable &&
      IsUpdateCf(table_)) {
    table_->GetTablet(&tablet_list_);
    for (std::size_t i = 0; i < tablet_list_.size(); ++i) {
      TabletPtr tablet = tablet_list_[i];
      // no other tablet transition procedure is allowed while tablet is
      // updating schema
      // Should be very carefully with tablet's transition lock
      if (!tablet->LockTransition()) {
        PROC_LOG(WARNING) << "abort update online table schema, tablet: " << tablet->GetPath()
                          << " in transition";
        for (std::size_t j = 0; j < i; ++j) {
          tablet = tablet_list_[j];
          tablet->UnlockTransition();
        }
        // 内存中回滚schem
        table_->AbortUpdate();
        EnterPhaseWithResponseStatus(kTableNotSupport, UpdateTablePhase::kEofPhase);
        return;
      }
      TabletMeta::TabletStatus status = tablet->GetStatus();
      if (status != TabletMeta::kTabletReady && status != TabletMeta::kTabletOffline) {
        PROC_LOG(WARNING) << "abort update online table schema, tablet: " << tablet->GetPath()
                          << " not in ready status, status: " << StatusCodeToString(status);
        for (std::size_t j = 0; j <= i; ++j) {
          tablet = tablet_list_[j];
          tablet->UnlockTransition();
        }
        table_->AbortUpdate();
        EnterPhaseWithResponseStatus(kTableNotSupport, UpdateTablePhase::kEofPhase);
        return;
      }
    }
  }
  SetNextPhase(UpdateTablePhase::kUpdateMeta);
}
```



### kTabletsSchemaSyncing:SyncTabletsSchemaHandler

```c++
void UpdateTableProcedure::SyncTabletsSchemaHandler(const UpdateTablePhase&) {
  if (sync_tablets_schema_) {
    return;
  }
  sync_tablets_schema_.store(true);
  PROC_LOG(INFO) << "begin sync tablets schema";
  tablet_sync_cnt_++;
  // No LoadTabletProcedure will be issued once the tablet falls into
  // kTabletOffline status while
  // UpdateTableProcedure is running, because UpdateTableProcedure has got the
  // tablet's TransitionLocks.
  // So UpdateTableProcedure should takes of those offline tablets by issue
  // LoadTabletProcedure for
  // those tablets at the right point of time.
  for (std::size_t i = 0; i < tablet_list_.size(); ++i) {
    TabletPtr tablet = tablet_list_[i];
    if (tablet->GetStatus() == TabletMeta::kTabletOffline) {
      
      // 记录offline的tablet，后续统一recover
      offline_tablets_.emplace_back(tablet);
      continue;
    }
    UpdateClosure done = std::bind(&UpdateTableProcedure::UpdateTabletSchemaCallback, this, tablet,
                                   0, _1, _2, _3, _4);
    // 同步通知ts更新schema
    NoticeTabletSchemaUpdate(tablet_list_[i], done);
    tablet_sync_cnt_++;
  }
  if (--tablet_sync_cnt_ == 0) {
    RecoverOfflineTablets();
    table_->CommitUpdate();
    PROC_VLOG(23) << "sync tablets schema finished";
    EnterPhaseWithResponseStatus(kMasterOk, UpdateTablePhase::kEofPhase);
  }
}



```

```c++
void UpdateTableProcedure::UpdateTabletSchemaCallback(TabletPtr tablet, int32_t retry_times, UpdateRequest* request, UpdateResponse* response, bool rpc_failed, int status_code) {
  std::unique_ptr<UpdateRequest> request_holder(request);
  std::unique_ptr<UpdateResponse> response_holder(response);
  StatusCode status = response_holder->status();
  TabletNodePtr node = tablet->GetTabletNode();

  if (tablet->GetStatus() == TabletMeta::kTabletOffline ||
      (!rpc_failed && status == kTabletNodeOk)) {
    if (tablet->GetStatus() == TabletMeta::kTabletOffline) {
      offline_tablets_.emplace_back(tablet);
    }
    // do not unlock offline tablets' TransitionLock. After all online tablets
    // UpdateTabletSchema RPC
    // has been collected, UpdateTableProcedure will issue LoadTabletProcedure
    // for those offline tablets
    // and those offline tablets will be locked until their LoadTabletProcedure
    // finished.
    else {
      tablet->UnlockTransition();
    }
    if (--tablet_sync_cnt_ == 0) {
      PROC_VLOG(23) << "sync tablets schema finished";
      
      // load offline tablet，然后再同步
      RecoverOfflineTablets();
      // 切换最新schema，清除就得schema
      table_->CommitUpdate();
      EnterPhaseWithResponseStatus(kMasterOk, UpdateTablePhase::kEofPhase);
    }
    return;
  }

  if (rpc_failed || status != kTabletNodeOk) {
    if (rpc_failed) {
      
    } else {
      
    }
    
    // 默认重试6w次
    if (retry_times > FLAGS_tera_master_schema_update_retry_times) {
      PROC_LOG(ERROR) << "[update] retry " << retry_times << " times, kick "
                      << tablet->GetServerAddr();
      // we ensure tablet's schema been updated by kickoff the hosting
      // tabletnode if all
      // UpdateTabletSchema RPC tries failed
      tablet->UnlockTransition();
      
      // 仍然失败 kick node
      MasterEnv().GetMaster()->TryKickTabletNode(tablet->GetServerAddr());
      if (--tablet_sync_cnt_ == 0) {
        RecoverOfflineTablets();
        table_->CommitUpdate();
        EnterPhaseWithResponseStatus(kMasterOk, UpdateTablePhase::kEofPhase);
      }
    } else {
      UpdateClosure done = std::bind(&UpdateTableProcedure::UpdateTabletSchemaCallback, this, tablet, retry_times + 1, _1, _2, _3, _4);
      ThreadPool::Task task =
          std::bind(&UpdateTableProcedure::NoticeTabletSchemaUpdate, this, tablet, done);
      MasterEnv().GetThreadPool()->DelayTask(FLAGS_tera_master_schema_update_retry_period * 1000,
                                             task);
    }
    return;
  }
}
```

