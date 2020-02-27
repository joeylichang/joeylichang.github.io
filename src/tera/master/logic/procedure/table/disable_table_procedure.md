# Master Disable Table 流程

Master侧设置table不可用分为一下几个阶段：

1. kPrepare：权限校验。
2. kDisableTable：table的状态机转换。
3. kUpdateMeta：更新元数据表，表状态为disable。
4. kDisableTablets：unload所有的tablet，有同步机制保证所有的tablet都会unload完毕。
5. kEofPhase：设置结束标志。

#### kDisableTable:DisableTableHandler && kUpdateMeta:UpdateMetaHandler

```c++
// 只是将table的状态机转换为kDisableTable，状态机内部没有其他逻辑
void DisableTableProcedure::DisableTableHandler(const DisableTablePhase&) {
  if (!table_->DoStateTransition(TableEvent::kDisableTable)) {
    PROC_LOG(WARNING) << table_->GetTableName() << " current state: " << table_->GetStatus()
                      << ", disable failed";
    EnterPhaseWithResponseStatus(static_cast<StatusCode>(table_->GetStatus()),
                                 DisableTablePhase::kEofPhase);
    return;
  }
  SetNextPhase(DisableTablePhase::kUpdateMeta);
}

// 打包元数据去更新，注意这里打包的元数据中table的状态已经是disable了
void DisableTableProcedure::UpdateMetaHandler(const DisableTablePhase&) {
  if (!update_meta_) {
    update_meta_.store(true);
    MetaWriteRecord record = PackMetaWriteRecord(table_, false);
    PROC_LOG(INFO) << "table: " << table_->GetTableName()
                   << " begin to update table disable info to meta";
    UpdateMetaClosure closure = std::bind(&DisableTableProcedure::UpdateMetaDone, this, _1);
    MasterEnv().BatchWriteMetaTableAsync(record, closure, FLAGS_tera_master_meta_retry_times);
  }
}

void DisableTableProcedure::UpdateMetaDone(bool succ) {
  if (!succ) {
    // disable failed because meta write fail, revert table's status to
    // kTableEnable
    PROC_CHECK(table_->DoStateTransition(TableEvent::kEnableTable));
    PROC_LOG(WARNING) << "fail to update meta";
    EnterPhaseWithResponseStatus(kMetaTabletError, DisableTablePhase::kEofPhase);
    return;
  }
  PROC_LOG(INFO) << "update disable table info to meta succ";
  EnterPhaseWithResponseStatus(kMasterOk, DisableTablePhase::kDisableTablets);
}
```

下面看一下table的状态转换图：

![tera_master_table_state_machine](../../../../../images/tera_master_table_state_machine.png)



#### kDisableTablets:DisableTabletsHandler

```c++
void DisableTableProcedure::DisableTabletsHandler(const DisableTablePhase&) {
  std::vector<TabletPtr> tablet_meta_list;
  table_->GetTablet(&tablet_meta_list);
  int in_transition_tablet_cnt = 0;
  for (uint32_t i = 0; i < tablet_meta_list.size(); ++i) {
    TabletPtr tablet = tablet_meta_list[i];
    if (tablet->GetStatus() == TabletMeta::kTabletDisable) {
      continue;
    }
    if (tablet->LockTransition()) {
      // unload之后的tablet的状态是kTabletOffline
      if (tablet->GetStatus() == TabletMeta::kTabletOffline ||
          tablet->GetStatus() == TabletMeta::kTabletLoadFail) {
        tablet->DoStateTransition(TabletEvent::kTableDisable);
        tablet->UnlockTransition();
        continue;
      }
      
      // 创建流程去unload tablet
      std::shared_ptr<Procedure> proc(new UnloadTabletProcedure(tablet, thread_pool_, false));
      MasterEnv().GetExecutor()->AddProcedure(proc);
      in_transition_tablet_cnt++;
    } else {
      in_transition_tablet_cnt++;
    }
  }
  PROC_VLOG(23) << "table: " << table_->GetTableName()
                << ", in transition num: " << in_transition_tablet_cnt;
  
  // 上层的调度模块会重复调用DisableTabletsHandler，知道in_transition_tablet_cnt == 0 才会跳转
  if (in_transition_tablet_cnt == 0) {
    SetNextPhase(DisableTablePhase::kEofPhase);
    return;
  }
}
```

tablet的状态转换图如下：

![tera_tablet_state_change](../../../../../images/tera_tablet_state_change.png)