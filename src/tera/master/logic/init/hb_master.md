# Master 心跳探测

Master 向Ts节点周期性（默认10s）发送一次心跳探测，心跳中带有权限、quta的信息，返回包中包括tablet、node的信息。核心内容在QueryTabletNode中。

### QueryTabletNode

```C++
void MasterImpl::QueryTabletNode() {
  // gc的逻辑后面会详细介绍gc的逻辑
  bool gc_query_enable = false;
  {
    MutexLock locker(&mutex_);
    if (!query_enabled_) {
      query_tabletnode_timer_id_ = kInvalidTimerId;
      return;
    }
    if (gc_query_enable_) {
      gc_query_enable_ = false;
      gc_query_enable = true;
    }
  }

  start_query_time_ = get_micros();
  std::vector<TabletNodePtr> tabletnode_array;
  tabletnode_manager_->GetAllTabletNodeInfo(&tabletnode_array);
  LOG(INFO) << "query tabletnodes: " << tabletnode_array.size() << ", id "
            << query_tabletnode_timer_id_;

  if (FLAGS_tera_stat_table_enabled) {
    stat_table_->OpenStatTable();
  }

  // 心跳的同步机制，query_pending_count_等是原子类型
  CHECK_EQ(query_pending_count_.Get(), 0);
  CHECK_EQ(update_auth_pending_count_.Get(), 0);
  CHECK_EQ(update_quota_pending_count_.Get(), 0);
  query_pending_count_.Inc();

  std::vector<TabletNodePtr>::iterator it = tabletnode_array.begin();
  for (; it != tabletnode_array.end(); ++it) {
    TabletNodePtr tabletnode = *it;
    if (tabletnode->state_ != kReady && tabletnode->state_ != kWaitKick) {
      VLOG(20) << "will not query tabletnode: " << tabletnode->addr_;
      continue;
    }
    query_pending_count_.Inc();
    update_auth_pending_count_.Inc();
    update_quota_pending_count_.Inc();
    // 异步向所有的节点发送探测消息
    QueryClosure done =
        std::bind(&MasterImpl::QueryTabletNodeCallback, this, tabletnode->addr_, _1, _2, _3, _4);
    // 会带有权限、quta、gc的信息发送给节点
    QueryTabletNodeAsync(tabletnode->addr_, FLAGS_tera_master_query_tabletnode_period,
                         gc_query_enable, done);
  }

  // 表示所有的心跳都发完并且返回了，这部分是一个补漏逻辑
  // 在QueryTabletNodeCallback内部也有，再次不详细介绍
  if (0 == query_pending_count_.Dec()) {
    LOG(INFO) << "query tabletnodes finish, id " << query_tabletnode_timer_id_
              << ", update auth failed ts count " << update_auth_pending_count_.Get()
              << ", update quota failed ts count " << update_quota_pending_count_.Get() << ", cost "
              << (get_micros() - start_query_time_) / 1000 << "ms.";
    (update_auth_pending_count_.Get() == 0)
        ? access_entry_->GetAccessUpdater().SyncUgiVersion(false)
        : access_entry_->GetAccessUpdater().SyncUgiVersion(true);
    // If ClearDeltaQuota failed, then means still need to sync version.
    if (update_quota_pending_count_.Get() == 0 && quota_entry_->ClearDeltaQuota()) {
      quota_entry_->SyncVersion(false);
    } else {
      quota_entry_->SyncVersion(true);
    }
    update_auth_pending_count_.Set(0);
    update_quota_pending_count_.Set(0);
    quota_entry_->RefreshClusterFlowControlStatus();
    quota_entry_->RefreshDfsHardLimit();
    {
      MutexLock locker(&mutex_);
      if (query_enabled_) {
        ScheduleQueryTabletNode();
      } else {
        query_tabletnode_timer_id_ = kInvalidTimerId;
      }
    }
    if (gc_query_enable) {
      DoTabletNodeGcPhase2();
    }
  }
}
```

接下来看一下QueryTabletNodeCallback，既心跳的返回包。

### QueryTabletNodeCallback

```C++
void MasterImpl::QueryTabletNodeCallback(std::string addr, QueryRequest *req, QueryResponse *res, bool failed, int error_code) {
  std::unique_ptr<QueryRequest> request{req};
  std::unique_ptr<QueryResponse> response{res};
  bool in_safemode = IsInSafeMode();
  int64_t query_callback_start = get_micros();
  const int64_t fuzzy_time = FLAGS_tera_master_query_tabletnode_period * 1000;
  TabletNodePtr node;
  if (!tabletnode_manager_->FindTabletNode(addr, &node)) {
    LOG(WARNING) << "fail to query: server down, id: " << request->sequence_id()
                 << ", server: " << addr;
  } else if (failed || response->status() != kTabletNodeOk) {
    // 重试10次 失败
    int32_t fail_count = node->IncQueryFailCount();
    if (fail_count >= FLAGS_tera_master_kick_tabletnode_query_fail_times) {
      LOG(ERROR) << kSms << "fail to query " << addr << " for " << fail_count << " times";
      TryKickTabletNode(addr);
    }
  } else {
    // update tablet meta
    uint32_t meta_num = response->tabletmeta_list().meta_size();
    std::map<tabletnode::TabletRange, int> tablet_map;
    for (uint32_t i = 0; i < meta_num; i++) {
      const TabletMeta &meta = response->tabletmeta_list().meta(i);
      const TabletCounter &counter = response->tabletmeta_list().counter(i);
      const std::string &table_name = meta.table_name();
      const std::string &key_start = meta.key_range().key_start();
      const std::string &key_end = meta.key_range().key_end();
      int64_t create_time = meta.create_time();
      uint64_t version = meta.version();

      TablePtr table;
      if (!tablet_manager_->FindTable(table_name, &table)) {
        continue;
      }
      // 新建的tablet，未完成
      if (create_time > 0 && create_time < table->CreateTime()) {
        continue;
      }

      std::vector<TabletPtr> tablets;
      // 在table中没有找到对应的tablet
      if (!table->FindOverlappedTablets(key_start, key_end, &tablets)) {
        continue;
      }

      // 可能正在split
      if (tablets.size() > 1) {
        bool splitted_tablet = true;
        for (uint32_t j = 0; j < tablets.size(); ++j) {
          if (version > tablets[j]->Version()) {
            LOG(FATAL) << "[query] tablet version error: " << tablets[j];
            // 如果是两个表示正在做
            splitted_tablet &= false;
          }
        }
        if (splitted_tablet) {
          TabletPtr stale_tablet(new StaleTablet(meta));
          // 表示这个是父tablet，尝试卸载它
          BindTabletToTabletNode(stale_tablet, node);
          TryUnloadTablet(stale_tablet);
        }
        continue;
      }

      CHECK_EQ(tablets.size(), 1u);
      TabletPtr tablet = tablets[0];
      if (version > 0 && version < tablet->Version()) {
        if (in_safemode) {
          continue;
        }
        TabletPtr stale_tablet(new StaleTablet(meta));
        LOG(WARNING) << "[query] try unload stale tablet: " << stale_tablet;
        BindTabletToTabletNode(stale_tablet, node);
        TryUnloadTablet(stale_tablet);
      }

      if (tablet->ReadyTime() >= start_query_time_) {
        VLOG(20) << "[query] ignore mutable tablet: " << meta.path() << " ["
                 << DebugString(key_start) << ", " << DebugString(key_end) << "] @ "
                 << meta.server_addr() << " status: " << StatusCodeToString(meta.status());
      } else if (tablet->GetKeyStart() != key_start || tablet->GetKeyEnd() != key_end) {
        LOG(ERROR) << "[query] range error tablet: " << meta.path() << " ["
                   << DebugString(key_start) << ", " << DebugString(key_end) << "] @ "
                   << meta.server_addr();
      } else if (tablet->GetPath() != meta.path()) {
        LOG(ERROR) << "[query] path error tablet: " << meta.path() << "] @ " << meta.server_addr()
                   << " should be " << tablet->GetPath();
      } else if (TabletMeta::kTabletReady != meta.status()) {
        LOG(ERROR) << "[query] status error tablet: " << meta.path() << "] @ " << meta.server_addr()
                   << "query status: " << StatusCodeToString(meta.status())
                   << " should be kTabletReady";
      } else if (tablet->GetServerAddr() != meta.server_addr()) {
        LOG(ERROR) << "[query] address tablet: " << meta.path() << " @ " << meta.server_addr()
                   << " should @ " << tablet->GetServerAddr();
      } else if (tablet->GetTable()->GetStatus() == kTableDisable) {
        LOG(INFO) << "table disabled: " << tablet->GetPath();
      } else {
        VLOG(20) << "[query] OK tablet: " << meta.path() << "] @ " << meta.server_addr();
        
        // 心跳包中需要更新的tablet信息
        tablet->SetUpdateTime(query_callback_start);
        tablet->UpdateSize(meta);
        tablet->SetCounter(counter);
        tablet->SetCompactStatus(meta.compact_status());
      }
    }

    // 更新ts，类似心跳收集的中的信息
    timeval update_time;
    gettimeofday(&update_time, NULL);
    TabletNode state;
    state.addr_ = addr;
    state.report_status_ = response->tabletnode_info().status_t();
    state.info_ = response->tabletnode_info();
    state.info_.set_addr(addr);
    state.load_ = response->tabletnode_info().load();
    state.persistent_cache_size_ = response->tabletnode_info().persistent_cache_size();
    state.data_size_ = 0;
    state.qps_ = 0;
    state.update_time_ = update_time.tv_sec * 1000 + update_time.tv_usec / 1000;
    // calculate data_size of tabletnode
    // count both Ready/OnLoad and OffLine tablet
    std::vector<TabletPtr> tablet_list;
    tablet_manager_->FindTablet(addr, &tablet_list, false);  // don't need disabled tables/tablets
    std::vector<TabletPtr>::iterator it;
    for (it = tablet_list.begin(); it != tablet_list.end(); ++it) {
      TabletPtr tablet = *it;
      // 存储节点中没有这个tablet
      if (tablet->UpdateTime() != query_callback_start) {
        if (tablet->GetStatus() == TabletMeta::kTabletUnloadFail && !in_safemode) {
          LOG(WARNING) << "[query] missed previous unload fail tablet, try move it: " << tablet;
          LOG(ERROR) << "[query] missed tablet, try move it: " << tablet;
          TryMoveTablet(tablet, tablet->GetTabletNode());
        }
        if (tablet->GetStatus() == TabletMeta::kTabletReady &&
            tablet->ReadyTime() + fuzzy_time < start_query_time_) {
          LOG(ERROR) << "[query] missed tablet, try move it: " << tablet;
          TryMoveTablet(tablet, tablet->GetTabletNode());
        }
      }

      TabletMeta::TabletStatus tablet_status = tablet->GetStatus();
      if (tablet_status == TabletMeta::kTabletReady ||
          tablet_status == TabletMeta::kTabletLoading ||
          tablet_status == TabletMeta::kTabletOffline) {
        state.data_size_ += tablet->GetDataSize();
        state.qps_ += tablet->GetQps();
        if (state.table_size_.find(tablet->GetTableName()) != state.table_size_.end()) {
          state.table_size_[tablet->GetTableName()] += tablet->GetDataSize();
          state.table_qps_[tablet->GetTableName()] += tablet->GetQps();
        } else {
          state.table_size_[tablet->GetTableName()] = tablet->GetDataSize();
          state.table_qps_[tablet->GetTableName()] = tablet->GetQps();
        }
      }
    }
    tabletnode_manager_->UpdateTabletNode(addr, state);
    node->ResetQueryFailCount();

    for (int32_t i = 0; i < response->tablet_background_errors_size(); i++) {
      const TabletBackgroundErrorInfo &background_error = response->tablet_background_errors(i);
      if (FLAGS_tera_stat_table_enabled) {
        stat_table_->RecordTabletCorrupt(background_error.tablet_name(),
                                         background_error.detail_info());
      }
    }
    VLOG(20) << "query tabletnode [" << addr
             << "], status_: " << StatusCodeToString(state.report_status_);
  }

  // 通知ts进行gc，master本地也要对tablt进行gc，后面详细介绍
  if (request->is_gc_query()) {
    for (int32_t i = 0; i < response->tablet_inh_file_infos_size(); i++) {
      const TabletInheritedFileInfo &tablet_inh_info = response->tablet_inh_file_infos(i);
      TablePtr table_ptr;
      if (tablet_manager_->FindTable(tablet_inh_info.table_name(), &table_ptr)) {
        table_ptr->GarbageCollect(tablet_inh_info);
      }
    }
  }

  // 版本校验
  if (response->has_version() &&
      access_entry_->GetAccessUpdater().IsSameVersion(response->version())) {
    update_auth_pending_count_.Dec();
  }

  // quta的版本校验
  if (response->has_quota_version() && quota_entry_->IsSameVersion(response->quota_version())) {
    update_quota_pending_count_.Dec();
  }

  // 表示所有发送的消息都回来了
  if (0 == query_pending_count_.Dec()) {
    LOG(INFO) << "query tabletnodes finish, id " << query_tabletnode_timer_id_
              << ", update auth failed ts count " << update_auth_pending_count_.Get()
              << ", update quota failed ts count " << update_quota_pending_count_.Get() << ", cost "
              << (get_micros() - start_query_time_) / 1000 << "ms.";
    // 表示权限信息的消息都回来了，测试master的权限信息的版本号+1
    (update_auth_pending_count_.Get() == 0)
        ? access_entry_->GetAccessUpdater().SyncUgiVersion(false)
        : access_entry_->GetAccessUpdater().SyncUgiVersion(true);
    // 表示quta信息的消息都回来了，测试master的权限信息的版本号+1
    if (update_quota_pending_count_.Get() == 0 && quota_entry_->ClearDeltaQuota()) {
      quota_entry_->SyncVersion(false);
    } else {
      quota_entry_->SyncVersion(true);
    }
    update_auth_pending_count_.Set(0);
    update_quota_pending_count_.Set(0);
    quota_entry_->RefreshClusterFlowControlStatus();
    quota_entry_->RefreshDfsHardLimit();
    {
      MutexLock locker(&mutex_);
      if (query_enabled_) {
        ScheduleQueryTabletNode();
      } else {
        query_tabletnode_timer_id_ = kInvalidTimerId;
      }
    }

    // 开启LB，后面详细介绍
    ScheduleLoadBalance();

    // gc部分介绍
    if (request->is_gc_query()) {
      DoTabletNodeGcPhase2();
    }
  }

  VLOG(20) << "query tabletnode finish " << addr << ", id " << query_tabletnode_timer_id_
           << ", callback cost " << (get_micros() - query_callback_start) / 1000 << "ms.";
}

```

