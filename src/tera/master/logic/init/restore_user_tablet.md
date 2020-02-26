# Master 加载UserTablet信息

Master加载完metatablet信息之后，会加载用户数据的tablet。

### RestoreUserTablet

```C++
void MasterImpl::RestoreUserTablet(const std::vector<TabletMeta> &report_meta_list) {
  std::vector<TabletMeta>::const_iterator meta_it = report_meta_list.begin();
  std::set<TablePtr> disabled_tables;
  for (; meta_it != report_meta_list.end(); ++meta_it) {
    const TabletMeta &meta = *meta_it;
    const std::string &table_name = meta.table_name();
    if (table_name == FLAGS_tera_master_meta_table_name) {
      continue;
    }
    const std::string &key_start = meta.key_range().key_start();
    const std::string &key_end = meta.key_range().key_end();
    const std::string &path = meta.path();
    const std::string &server_addr = meta.server_addr();
    TabletNodePtr node = tabletnode_manager_->FindTabletNode(meta.server_addr(), NULL);
    CompactStatus compact_status = meta.compact_status();
    TabletMeta::TabletStatus status = meta.status();

    TabletPtr tablet;
    /*
     * FindTablet ： TabletManager -> all_tables_ -> <start, end> 元数据中加载的信息
     * Verify ： table_name、key_range、path、server_addr进行校验
     */
    if (!tablet_manager_->FindTablet(table_name, key_start, &tablet) ||
        !tablet->Verify(table_name, key_start, key_end, path, server_addr)) {
      LOG(INFO) << "unload unexpected table: " << path << ", server: " << server_addr;
      TabletMeta unknown_meta = meta;
      unknown_meta.set_status(TabletMeta::kTabletReady);
      TabletPtr unknown_tablet(new UnknownTablet(unknown_meta));
      
      // tablet 绑定 node addr
      BindTabletToTabletNode(unknown_tablet, node);
      // master 调度prodecure 去卸载它，后续介绍里面流程
      TryUnloadTablet(unknown_tablet);
    } else {
      BindTabletToTabletNode(tablet, node);
      
      /*
       * 后续是tablet根据状态进行的处理逻辑
       */
      
      // tablets of a table may be partially disabled before master deaded, so
      // we need try disable
      // the table once more on master restarted
      if (tablet->GetTable()->GetStatus() == kTableDisable) {
        disabled_tables.insert(tablet->GetTable());
        continue;
      }
      tablet->UpdateSize(meta);
      tablet->SetCompactStatus(compact_status);
      // if the actual status of a tablet reported by ts is unloading, try move
      // it to make sure it be loaded finally
      if (status == TabletMeta::kTabletUnloading || status == TabletMeta::kTabletUnloading2) {
        tablet->SetStatus(TabletMeta::kTabletReady);
        TryMoveTablet(tablet);
        continue;
      }
      if (status == TabletMeta::kTabletReady) {
        tablet->SetStatus(TabletMeta::kTabletReady);
      }
      // treat kTabletLoading reported from TS as kTabletOffline, thus we will
      // try to load it in subsequent lines
    }
  }

  std::vector<TabletPtr> all_tablet_list;
  tablet_manager_->ShowTable(NULL, &all_tablet_list);
  std::vector<TabletPtr>::iterator it;
  for (it = all_tablet_list.begin(); it != all_tablet_list.end(); ++it) {
    TabletPtr tablet = *it;
    if (tablet->GetTableName() == FLAGS_tera_master_meta_table_name) {
      continue;
    }
    // there may exists in transition tablets here as we may have a
    // MoveTabletProcedure for it
    // if its reported status is unloading
    if (tablet->InTransition()) {
      LOG(WARNING) << "give up restore in transition tablet, tablet: " << tablet;
      continue;
    }
    const std::string &server_addr = tablet->GetServerAddr();
    if (tablet->GetStatus() == TabletMeta::kTabletReady) {
      VLOG(8) << "READY Tablet, " << tablet;
      continue;
    }
    if (tablet->GetStatus() != TabletMeta::kTabletOffline) {
      LOG(ERROR) << kSms << "tablet " << tablet
                 << ", unexpected status: " << StatusCodeToString(tablet->GetStatus());
      continue;
    }
    if (tablet->GetTable()->GetStatus() == kTableDisable) {
      disabled_tables.insert(tablet->GetTable());
      continue;
    }

    // 加载tablet
    TabletNodePtr node;
    if (server_addr.empty()) {
      VLOG(8) << "OFFLINE Tablet with empty addr, " << tablet;
    } else if (!tabletnode_manager_->FindTabletNode(server_addr, &node)) {
      VLOG(8) << "OFFLINE Tablet of Dead TS, " << tablet;
    } else if (node->state_ == kReady) {
      VLOG(8) << "OFFLINE Tablet of Alive TS, " << tablet;
      TryLoadTablet(tablet, node);
    } else {
      // Ts not response, try load it
      TryLoadTablet(tablet, node);
      VLOG(8) << "UNKNOWN Tablet of No-Response TS, try load it" << tablet;
    }
  }
  
  // 删除disable的tablet
  for (auto &table : disabled_tables) {
    if (table->LockTransition()) {
      DisableAllTablets(table);
    }
  }
}
```

上述流程中又很多涉及调度prodecure会再后续介绍。

tablet的状态转换图，如下所示：

![tera_tablet_state_change](../../../../../images/tera_tablet_state_change.png)
