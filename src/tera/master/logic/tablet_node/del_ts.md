# Master Delete TabletNode

### 删除节点的时机

Master 检测到zk，ts注册目录的变化的时候，会全量获取ts注册目录的子节点信息，与Master内存中的数据进行对比，多的就加入，少的就删除。

### 源码

```c++
void MasterImpl::DeleteTabletNode(const std::string &tabletnode_addr, const std::string &uuid) {
  // abnormal_node 防止节点频繁抖动，见之前介绍文章
  abnormal_node_mgr_->RecordNodeDelete(tabletnode_addr, get_micros() / 1000000);
  
  // 在tabletnode_manager_中删除节点，后续介绍
  TabletNodePtr node = tabletnode_manager_->DelTabletNode(tabletnode_addr);
  if (!node || node->uuid_ != uuid) {
    LOG(INFO) << "invalid node and uuid: addr: " << tabletnode_addr << ", uuid: " << uuid;
    return;
  }

  // possible status: running, readonly, wait.
  if (GetMasterStatus() == kOnWait) {
    return;
  }

  std::vector<TabletPtr> tablet_list;
  tablet_manager_->FindTablet(tabletnode_addr, &tablet_list, false);

  // 如果meta在上面县搬迁meta_tablet_，因为搬迁需要修改元信息
  if (meta_tablet_->GetTabletNode() && meta_tablet_->GetTabletNode()->uuid_ == uuid) {
    LOG(INFO) << " try move meta tablet immediately: ";
    TryMoveTablet(meta_tablet_);
    auto pend = std::remove(tablet_list.begin(), tablet_list.end(),
                            std::dynamic_pointer_cast<Tablet>(meta_tablet_));
    tablet_list.erase(pend, tablet_list.end());
  }

  // 判断是否可以进入safe_mode状态，10%的tablet故障
  bool in_safemode = TryEnterSafeMode();
  // 如果是safe_mode则进入kTsDelayOffline，否则tablet进入kTsOffline
  TabletEvent event = in_safemode ? TabletEvent::kTsOffline : TabletEvent::kTsDelayOffline;
  for (auto it = tablet_list.begin(); it != tablet_list.end(); ++it) {
    TabletPtr tablet = *it;
    if ((tablet->GetTabletNode() && tablet->GetTabletNode()->uuid_ == uuid) &&
        tablet->LockTransition()) {
      tablet->DoStateTransition(event);
      tablet->UnlockTransition();
    }
  }
  if (in_safemode) {
    LOG(WARNING) << "master is in safemode, will not recover user tablet at ts: "
                 << tabletnode_addr;
    return;
  }
  
  // 60s或者30s
  int64_t wait_time = FLAGS_tera_master_tabletnode_timeout;
  wait_time = wait_time ? wait_time : 3 * FLAGS_tera_master_query_tabletnode_period;
  // 搬迁tablet的逻辑，见相关文章
  ThreadPool::Task task =
      std::bind(&MasterImpl::MoveTabletOnDeadTabletNode, this, tablet_list, node);
  int64_t reconnect_timeout_task_id = thread_pool_->DelayTask(wait_time, task);
  
  // 加入重连队列，只有重新add节点才会pop出来，要是节点永久删除了，重连队列一直有数据
  tabletnode_manager_->WaitTabletNodeReconnect(tabletnode_addr, uuid, reconnect_timeout_task_id);
}
```



```c++
TabletNodePtr TabletNodeManager::DelTabletNode(const std::string& addr) {
  TabletNodePtr state(nullptr);

  // 在tabletnode_list_中删除
  MutexLock lock(&mutex_);
  TabletNodeList::iterator it = tabletnode_list_.find(addr);
  if (it == tabletnode_list_.end()) {
    LOG(ERROR) << "tabletnode [" << addr << "] does not exist";
    return state;
  }
  state = it->second;
  // 设置kZkSessionTimeout时间，进入offline状态
  state->DoStateTransition(NodeEvent::kZkSessionTimeout);
  // state->SetState(kOffLine, NULL);
  tabletnode_list_.erase(it);

  // delete node may block, so we'd better release the mutex before that
  LOG(INFO) << "delete tabletnode: " << addr << ", uuid: " << state->uuid_;
  return state;
}
```

