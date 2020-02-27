# Master Merge Tablet 流程

Master侧合并Tablet的流程，如下：

1. 合并的两个tablet必须是kTabletReady状态，并且key连续。
2. unload两个tablet。
3. 校验是否unload成功，策略类似spilt流程（校验log文件，失败则加载2个源tablet）。
4. 更新元数据，添加一天新的tablet元数据。
5. 更新成功之后，更新内存数据。
6. 加载新的tablet'。

### kUpdateMeta

```c++
void MergeTabletProcedure::UpdateMeta() {
  std::vector<MetaWriteRecord> records;
  PackMetaWriteRecords(tablets_[0], true, records);
  PackMetaWriteRecords(tablets_[1], true, records);

  TabletMeta new_meta;
  // <tablets_[1], tablets_[0]>
  
  // 填充new_meta
  if (tablets_[0]->GetKeyStart() == tablets_[1]->GetKeyEnd() && tablets_[0]->GetKeyStart() != "") {
    tablets_[1]->ToMeta(&new_meta);
    new_meta.mutable_key_range()->set_key_end(tablets_[0]->GetKeyEnd());
    new_meta.clear_parent_tablets();
    new_meta.add_parent_tablets(leveldb::GetTabletNumFromPath(tablets_[1]->GetPath()));
    new_meta.add_parent_tablets(leveldb::GetTabletNumFromPath(tablets_[0]->GetPath()));
  }
  // <tablets_[0], tablets_[1]>
  else if (tablets_[0]->GetKeyEnd() == tablets_[1]->GetKeyStart()) {
    tablets_[0]->ToMeta(&new_meta);
    new_meta.mutable_key_range()->set_key_end(tablets_[1]->GetKeyEnd());
    new_meta.clear_parent_tablets();
    new_meta.add_parent_tablets(leveldb::GetTabletNumFromPath(tablets_[0]->GetPath()));
    new_meta.add_parent_tablets(leveldb::GetTabletNumFromPath(tablets_[1]->GetPath()));
  } else {
    PROC_LOG(FATAL) << "tablet range error, cannot be merged" << tablets_[0] << ", " << tablets_[1];
  }

  new_meta.set_status(TabletMeta::kTabletOffline);
  std::string new_path = leveldb::GetChildTabletPath(tablets_[0]->GetPath(),
                                                     tablets_[0]->GetTable()->GetNextTabletNo());
  new_meta.set_path(new_path);
  new_meta.set_size(tablets_[0]->GetDataSize() + tablets_[1]->GetDataSize());
  uint64_t version = tablets_[0]->Version() > tablets_[1]->Version() ? tablets_[0]->Version()
                                                                     : tablets_[1]->Version();
  new_meta.set_version(version + 1);
  merged_.reset(new Tablet(new_meta, tablets_[0]->GetTable()));

  dest_node_ =
      (tablets_[0]->GetDataSize() > tablets_[1]->GetDataSize() ? tablets_[0]->GetTabletNode()
                                                               : tablets_[1]->GetTabletNode());
  PackMetaWriteRecords(merged_, false, records);
  UpdateMetaClosure done = std::bind(&MergeTabletProcedure::MergeUpdateMetaDone, this, _1);
  PROC_LOG(INFO) << "[merge] update meta, tablet: [" << tablets_[0]->GetPath() << ", "
                 << tablets_[1]->GetPath() << "]";
  // update meta table asynchronously until meta write ok.
  MasterEnv().BatchWriteMetaTableAsync(records, done, -1);
}
```



### kLoadMergedTablet

```c++
void MergeTabletProcedure::LoadMergedTabletPhaseHandler(const MergeTabletPhase&) {
  if (!load_proc_) {
    // 注意merged_ 是新的tablet
    load_proc_.reset(new LoadTabletProcedure(merged_, dest_node_, thread_pool_, true));
    PROC_LOG(INFO) << "Generate LoadTablet SubProcedure: " << load_proc_->ProcId()
                   << "merged: " << merged_ << ", destnode: " << dest_node_->GetAddr();
    MasterEnv().GetExecutor()->AddProcedure(load_proc_);
  }
  SetNextPhase(MergeTabletPhase::kEofPhase);
}
```

