# Master Delete TabletNode

### kick 节点的含义

kick状态的节点，其实是一种异常状态，节点在zk注册的节点还在，还不能进行删除。但是Master与节点进行交互，比如unload_tablet、加载meta_tablet、心跳超过重试次数仍未返回结果（10次）、初始化收集节点信息超过重试次数（10次）仍未返回信息时，节点被kick。

### 源码

```c++
bool MasterImpl::TryKickTabletNode(TabletNodePtr node) {
  // concurrently kicking is not allowed
  std::lock_guard<std::mutex> lock(kick_mutex_);
  
  // 节点处于kKicked或者offline状态则为kickked状态
  if (node->NodeKicked()) {
    VLOG(6) << "node has already been kicked, addr: " << node->GetAddr()
            << ", uuid: " << node->GetId()
            << ", state: " << StatusCodeToString((StatusCode)node->GetState());
    return true;
  }
  if (!FLAGS_tera_master_kick_tabletnode_enabled) {
    VLOG(6) << "kick is disabled,  addr: " << node->GetAddr() << ", uuid: " << node->GetId();
    return false;
  }

  if (IsInSafeMode()) {
    VLOG(6) << "cancel kick ts, master is in safemode, addr: " << node->GetAddr()
            << ", uuid: " << node->GetId();
    return false;
  }

  if (!node->DoStateTransition(NodeEvent::kPrepareKickTs)) {
    return false;
  }

  // 如果处于safe_mode不可以kick节点
  double tablet_locality_ratio = LiveNodeTabletRatio();
  LOG(INFO) << "tablet locality ratio: " << tablet_locality_ratio;
  if (tablet_locality_ratio < FLAGS_tera_safemode_tablet_locality_ratio) {
    node->DoStateTransition(NodeEvent::kCancelKickTs);
    LOG(WARNING) << "tablet live ratio will fall to: " << tablet_locality_ratio
                 << ", cancel kick ts: " << node->GetAddr();
    return false;
  }

  // 在zk，"/tear/kick/"目录下添加节点
  if (!zk_adapter_->KickTabletServer(node->addr_, node->uuid_)) {
    LOG(WARNING) << "kick tabletnode fail, node: " << node->addr_ << "," << node->uuid_;
    node->DoStateTransition(NodeEvent::kCancelKickTs);
    // revert node status;
    return false;
  }
  // 节点进入kicked状态
  node->DoStateTransition(NodeEvent::kZkKickNodeCreated);
  return true;
}
```

