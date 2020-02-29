# Master Add TabletNode

### 加入节点时机

TabletNode在启动时会在zk注册节点，Masterwtch到变化，会更新节点完成节点的加入。

在初始化阶段，会先获取zk注册的信息，加入节点，然后Query请求（心跳探测）根据信息去回填TableNodeManager中相应的元信息。

### 源码解析

```c++
void MasterImpl::AddTabletNode(const std::string &tabletnode_addr,
                               const std::string &tabletnode_uuid) {
  // abnormal_node 防止节点频繁抖动，在之前的文章中有介绍
  if (abnormal_node_mgr_->IsAbnormalNode(tabletnode_addr, tabletnode_uuid)) {
    LOG(WARNING) << abnormal_node_mgr_->GetNodeInfo(tabletnode_addr);
    return;
  }

  // tabletnode_manager_中加入节点，但是元信息还是空，后面介绍详细逻辑
  TabletNodePtr node = tabletnode_manager_->AddTabletNode(tabletnode_addr, tabletnode_uuid);
  if (!node) {
    return;
  }
  CHECK(node->GetState() == kReady);
  
  // update tabletnode info
  timeval update_time;
  gettimeofday(&update_time, NULL);
  TabletNode state;
  state.addr_ = tabletnode_addr;
  state.report_status_ = kTabletNodeReady;
  state.info_.set_addr(tabletnode_addr);
  state.data_size_ = 0;
  state.qps_ = 0;
  state.update_time_ = update_time.tv_sec * 1000 + update_time.tv_usec / 1000;

  // 回填元信息
  tabletnode_manager_->UpdateTabletNode(tabletnode_addr, state);

  // If all tabletnodes restart in one zk callback,
  // master will not enter restore/wait state;
  // meta table must be scheduled to load from here.
  // 下面是tablet的逻辑，需要更新元数据，所以需要保证元数据必须ready
  // TryLoadTablet见tablet加载逻辑的介绍
  if (meta_tablet_->GetStatus() == TabletMeta::kTabletOffline) {
    TryLoadTablet(meta_tablet_);
  }

  // 如果节点在重连队列，需要cancel
  int64_t reconn_taskid = tabletnode_manager_->PopTabletNodeReconnectTaskID(tabletnode_addr);
  if (reconn_taskid > 0) {
    thread_pool_->CancelTask(reconn_taskid);
    LOG(INFO) << "tabletnode reconnected, cancel reconn tiemout task, id: " << reconn_taskid;
  }

  // load offline tablets
  // update tabletnode
  // 加载offline的tablet，注意：tablet的offline状态不表示无用，disable状态才是
  std::vector<TabletPtr> tablet_list;
  tablet_manager_->FindTablet(tabletnode_addr, &tablet_list, false);  // need disabled table/tablets
  std::vector<TabletPtr>::iterator it = tablet_list.begin();
  for (; it != tablet_list.end(); ++it) {
    TabletPtr tablet = *it;
    if (tablet->LockTransition()) {
      if (tablet->GetStatus() == TabletMeta::kTabletDelayOffline) {
        tablet->DoStateTransition(TabletEvent::kTsOffline);
      }
      if (tablet->GetStatus() != TabletMeta::kTabletOffline) {
        tablet->UnlockTransition();
        LOG(WARNING) << "tablet cannot deal TsOffline event, tablet:  " << tablet;
        continue;
      }
      std::shared_ptr<Procedure> load(new LoadTabletProcedure(tablet, node, thread_pool_.get()));
      if (MasterEnv().GetExecutor()->AddProcedure(load) == 0) {
        LOG(WARNING) << "add to procedure_executor fail, may duplicated procid: " << load->ProcId();
        tablet->UnlockTransition();
      }
    }
  }
  // safemode must be manual checked and manual leave
}
```



```c++
TabletNodePtr TabletNodeManager::AddTabletNode(const std::string& addr, const std::string& uuid) {
  MutexLock lock(&mutex_);
  TabletNodePtr null_ptr;
  // 从tabletnode_list_取出node
  std::pair<TabletNodeList::iterator, bool> ret =
      tabletnode_list_.insert(std::pair<std::string, TabletNodePtr>(addr, null_ptr));
  TabletNodePtr& state = ret.first->second;
  // already has one TS at the same IP:PORT addr, return the existing
  // TabletNodePtr
  // 表示已经存在
  if (!ret.second) {
    TabletNodePtr existing_node = ret.first->second;
    LOG(ERROR) << "tabletnode [" << addr << " exist, existing uuid: " << existing_node->uuid_
               << ", to be added uuid: " << uuid;
    return existing_node;
  } else {
    LOG(INFO) << "add tabletnode : " << addr << ", id : " << uuid;
    state.reset(new TabletNode(addr, uuid));
  }
  // kReady represent heartbeat status
  // state->SetState(kReady, NULL);
  // 进入kZkNodeCreated状态
  state->DoStateTransition(NodeEvent::kZkNodeCreated);
  // 信号量，用于move tablet时选择节点的同步
  tabletnode_added_.Broadcast();
  return state;
}
```



### TabletNode状态机

![tera_tabke_node_state](../../../../../images/tera_tabke_node_state.png)

注意：节点处理在线，不在线还有kick的状态，kKicked表示节点下线。