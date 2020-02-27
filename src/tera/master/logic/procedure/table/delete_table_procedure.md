# Master Delete Table 流程

删除表的流程和创建表的代码架构相似，也是分位四个阶段：

1. kPrepare：权限校验。
2. kDeleteTable：tablet状态校验（必须是disable状态的tablet），元数据整理，追加写（删除标记）到meta_tablet。
3. kUpdateMeta：更新源数据表，并在内存中删除数据。
4. kEofPhase：结束标记设定。

### kDeleteTable:DeleteTableHandler

```c++
void DeleteTableProcedure::DeleteTableHandler(const DeleteTablePhase&) {
  std::vector<TabletPtr> tablets;
  table_->GetTablet(&tablets);
  // 状态校验
  for (size_t i = 0; i < tablets.size(); ++i) {
    TabletPtr tablet = tablets[i];
    if (tablet->GetStatus() != TabletMeta::kTabletDisable) {
      PROC_LOG(WARNING) << "tablet: " << tablet << " not in disabled status, "
                        << StatusCodeToString(tablet->GetStatus());
      EnterPhaseWithResponseStatus(StatusCode(tablet->GetStatus()), DeleteTablePhase::kEofPhase);
      return;
    }
    // 元数据打包，设置删除标记
    PackMetaWriteRecords(tablet, true, meta_records_);
  }
  if (!table_->DoStateTransition(TableEvent::kDeleteTable)) {
    PROC_LOG(WARNING) << "table: " << table_->GetTableName()
                      << ", current status: " << StatusCodeToString(table_->GetStatus());
    EnterPhaseWithResponseStatus(kTableNotSupport, DeleteTablePhase::kEofPhase);
    return;
  }
  // 元数据打包，设置删除标记
  PackMetaWriteRecords(table_, true, meta_records_);

  // delete quota setting store in meta table
  quota::MasterQuotaHelper::PackDeleteQuotaRecords(table_->GetTableName(), meta_records_);

  SetNextPhase(DeleteTablePhase::kUpdateMeta);
}
```



### kUpdateMeta:UpdateMetaHandler

```c++
void DeleteTableProcedure::UpdateMetaHandler(const DeleteTablePhase&) {
  if (update_meta_) {
    return;
  }
  update_meta_.store(true);
  PROC_LOG(INFO) << "table: " << table_->GetTableName() << "begin to update meta";
  UpdateMetaClosure closure = std::bind(&DeleteTableProcedure::UpdateMetaDone, this, _1);
  MasterEnv().BatchWriteMetaTableAsync(meta_records_, closure, FLAGS_tera_master_meta_retry_times);
}


void DeleteTableProcedure::UpdateMetaDone(bool succ) {
  if (!succ) {
    PROC_LOG(WARNING) << "table: " << table_->GetTableName() << " update meta fail";
    EnterPhaseWithResponseStatus(kMetaTabletError, DeleteTablePhase::kEofPhase);
    return;
  }
  PROC_LOG(INFO) << "table: " << table_->GetTableName() << " update meta succ";
  if (!MasterEnv().GetQuotaEntry()->DelRecord(table_->GetTableName())) {
    PROC_LOG(WARNING) << "table: " << table_->GetTableName()
                      << " delete master memory quota cache failed";
  }
  StatusCode code;
  // 删除内存中的表数据
  MasterEnv().GetTabletManager()->DeleteTable(table_->GetTableName(), &code);
  EnterPhaseWithResponseStatus(kMasterOk, DeleteTablePhase::kEofPhase);
}
```

