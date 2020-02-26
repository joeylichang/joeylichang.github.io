# Master 收集mste_tablet信息逻辑

### 主要步骤

1. Master根据zk中的ts数据，先将节点加入tablenode_manager，tabletnode_manager_->AddTabletNode(addr, uuid);
2. Master根据zk中的ts信息，并发异步的向ts发送请求获取节点信息。
   1. 请求的超时时间是30s，重试10次，重试间隔3s，总超时时间是 11 * 30s + 10 * 3s = 360s = 6min。
   2. 重试都失败之后对节点进行kick（TryKickTabletNode）。
   3. 并发异步的同步机制是信号量。
3. 收集之后的信息更新 tabletnode_manager_的TabletNode（UpdateTabletNode）。

### 更新的TabletNode数据

```c++
struct TabletNode {
  mutable Mutex mutex_;
  std::string addr_;    // 地址
  std::string uuid_;    // 节点uuid 位数包含10位的版本号
  NodeState state_;     // kReady/kOffline/kPendingOffline/kOnKick/kWaitKick/kKicked
  int64_t timestamp_;

  /* kTabletNodeNotInited = 22;
     kTabletNodeIsBusy = 23;
     kTabletNodeIsIniting = 24;
     kTabletNodeIsReadonly = 25;
     kTabletNodeIsRunning = 29;
   */
  StatusCode report_status_;     
  TabletNodeInfo info_;               // 节点的统计信息（读写流量、cpu等）
  uint64_t data_size_;                // tablet大小和
  uint64_t qps_;                      // 第一次收集时清为0
  uint64_t load_;                     // 节点负载
  uint64_t persistent_cache_size_;    // ssdb的大小
  uint64_t update_time_;              // 收集时的时间戳
  std::map<std::string, uint64_t> table_size_;   // table 级别的大小
  std::map<std::string, uint64_t> table_qps_;    // 第一次收集时为0

  struct MutableCounter {
    uint64_t read_pending_;
    uint64_t write_pending_;
    uint64_t scan_pending_;
    uint64_t row_read_delay_;  // micros

    MutableCounter() { memset(this, 0, sizeof(MutableCounter)); }
  };
  MutableCounter average_counter_;           // 当前收集值权重为2，前值权重为1
};
```

### 收集失败的节点

```C++
bool MasterImpl::TryKickTabletNode(TabletNodePtr node) {
  // concurrently kicking is not allowed
  std::lock_guard<std::mutex> lock(kick_mutex_);
  if (node->NodeKicked()) {
    return true;
  }
  if (!FLAGS_tera_master_kick_tabletnode_enabled) {
    return false;
  }

  if (IsInSafeMode()) {
    return false;
  }

  if (!node->DoStateTransition(NodeEvent::kPrepareKickTs)) {
    return false;
  }

  double tablet_locality_ratio = LiveNodeTabletRatio();
  if (tablet_locality_ratio < FLAGS_tera_safemode_tablet_locality_ratio) {
    node->DoStateTransition(NodeEvent::kCancelKickTs);
    return false;
  }

  //  创建"/tera/kick/uuid"节点
  if (!zk_adapter_->KickTabletServer(node->addr_, node->uuid_)) {
    LOG(WARNING) << "kick tabletnode fail, node: " << node->addr_ << "," << node->uuid_;
    node->DoStateTransition(NodeEvent::kCancelKickTs);
    // revert node status;
    return false;
  }
  
  // 节点状态转变为kKicked，并没有从tablenode_manger中删除
  node->DoStateTransition(NodeEvent::kZkKickNodeCreated);
  return true;
}
```

可以看到上述逻辑中并没有将节点从集群删除，而是在zk中添加到一个新的目录中，并对节点的状态进行了更新，这里就有必要看一下节点的状态机。

![tera_tabke_node_state](../../../../../images/tera_tabke_node_state.png)
