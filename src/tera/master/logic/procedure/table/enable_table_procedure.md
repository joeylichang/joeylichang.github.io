# Master Enable Table 流程

Master 侧enable table流程分位一下5个阶段进行：

1. kPrepare：权限校验。
2. kEnableTable：转换table的状态机为kEnableTable。
3. kUpdateMeta：更新table的状态到源数据表。
4. kEnableTablets：load tablts。
5. kEofPhase：设置结束标志位。

Enable流程与Disable流程相似，这里重点介绍kEnableTablets。

### kEnableTablets:EnableTabletsHandler

```c++
void EnableTableProcedure::EnableTabletsHandler(const EnableTablePhase&) {
  std::vector<TabletPtr> tablets;
  table_->GetTablet(&tablets);
  for (std::size_t i = 0; i < tablets.size(); ++i) {
    TabletPtr tablet = tablets[i];
    PROC_CHECK(tablet->LockTransition()) << tablet->GetPath() << " in another tansition";
    PROC_CHECK(tablet->DoStateTransition(TabletEvent::kTableEnable))
        << tablet->GetPath() << ", current status: " << tablet->GetStatus();
    
    // 创建load_tablets流程，注意可能没有全部load完，enable流程已经结束了
    std::shared_ptr<Procedure> proc(
        new LoadTabletProcedure(tablet, tablet->GetTabletNode(), thread_pool_));
    MasterEnv().GetExecutor()->AddProcedure(proc);
  }
  SetNextPhase(EnableTablePhase::kEofPhase);
}

```

