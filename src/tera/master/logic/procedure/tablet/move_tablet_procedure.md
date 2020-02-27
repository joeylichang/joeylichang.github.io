

# Master Move Tablet 流程

Master侧搬迁tablet的流程中，与tablet的初始状态相关。

1. kTabletReady || kTabletUnloadFail：源节点执行 UnloadTabletPhaseHandler。
2. kTabletOffline || kTabletLoadFail：目标节点执行kLoadTablet。

### UnloadTabletPhaseHandler

```c++
void MoveTabletProcedure::UnloadTabletPhaseHandler(const MoveTabletPhase&) {
  
  // 表示uload 流程没有结束，会等待其执行完
  if (!unload_proc_) {
    PROC_LOG(INFO) << "MoveTablet: Unload: " << tablet_;
    unload_proc_.reset(new UnloadTabletProcedure(tablet_, thread_pool_, true));
    MasterEnv().GetExecutor()->AddProcedure(unload_proc_);
  }
  
  // 循环调用
  if (!unload_proc_->Done()) {
    return;
  }
  if (tablet_->GetStatus() != TabletMeta::kTabletOffline) {
    // currently if unload fail, we directly abort the MoveTabletProcedure. U
    // should also
    // notice that if master_kick_tabletnode is enabled, we will never fall into
    // unload
    // fail position because we can always unload the tablet succ by kick off
    // the TS
    // TODO: if dfs directory lock is enabled, we can enter LOAD_TABLET phase
    // directly
    // as directory lock ensures we can avoid multi-load problem
    SetNextPhase(MoveTabletPhase::kEofPhase);
    return;
  }
  SetNextPhase(MoveTabletPhase::kLoadTablet);
}
```



### LoadTabletPhaseHandler

```c++
void MoveTabletProcedure::LoadTabletPhaseHandler(const MoveTabletPhase&) {
  if (!load_proc_) {
    tablet_->IncVersion();
    
    // 走load逻辑
    load_proc_.reset(new LoadTabletProcedure(tablet_, dest_node_, thread_pool_, true));
    PROC_LOG(INFO) << "MoveTablet: generate async LoadTabletProcedure: " << load_proc_->ProcId()
                   << "tablet " << tablet_;
    MasterEnv().GetExecutor()->AddProcedure(load_proc_);
  }
  SetNextPhase(MoveTabletPhase::kEofPhase);
}
```

