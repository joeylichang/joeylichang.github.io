# Master 初始化

master 在初始化函数中创建成员变量之后，主要的逻辑InitAsync中，包括zk初始化、TabletNode Restore、TableRestore、UserTabletRestore、LeaveSafeMode、可用性探测。

### InitAsync

```c++
void MasterImpl::InitAsync() {
  std::string meta_tablet_addr;
  std::map<std::string, std::string> tabletnode_list;
  bool safe_mode = false;

  // Make sure tabletnode_list will not change
  // during restore process.
  MutexLock lock(&tabletnode_mutex_);

  // 初始化zk，创建目录节点、获取ts全量信息
  while (!zk_adapter_->Init(&meta_tablet_addr, &tabletnode_list, &safe_mode)) {
    LOG(ERROR) << kSms << "zookeeper error, please check!";
  }

  DoStateTransition(MasterEvent::kGetMasterLock);

  // 加载数据
  Restore(tabletnode_list);

  // 定时（60s）从延迟队列中获取节点加入集群
  ScheduleDelayAddNode();
}

bool MasterImpl::Restore(const std::map<std::string, std::string> &tabletnode_list) {
  tabletnode_mutex_.AssertHeld();
  CHECK(!restored_);

  if (tabletnode_list.size() == 0) {
    DoStateTransition(MasterEvent::kNoAvailTs);
    LOG(ERROR) << kSms << "no available tabletnode";
    return false;
  }

  // 收集ts信息并更新，获取ts上面全量的tablet
  std::vector<TabletMeta> tablet_list;
  CollectAllTabletInfo(tabletnode_list, &tablet_list);

  // 加载tablet的原信息
  if (!RestoreMetaTablet(tablet_list)) {
    DoStateTransition(MasterEvent::kNoAvailTs);
    return false;
  }

  DoStateTransition(MasterEvent::kMetaRestored);

  user_manager_->SetupRootUser();

  // 加载UserTablet
  RestoreUserTablet(tablet_list);

  // tablet 的存活率大于90% 即可
  TryLeaveSafeMode();
  // 节点可用性探测，心跳貌似是ts主动上报的，这里只进行了统计梳理，默认60s
  EnableAvailabilityCheck();
  // 定时更新统计信息（table级别），默认10s
  RefreshTableCounter();

  // restore success
  restored_ = true;
  return true;
}
```



