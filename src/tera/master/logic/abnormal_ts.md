# 节点恢复（防抖动）

TabletServer 掉线再恢复，可以通过添加节点进行处理，但是这远远不够。比如节点由于网络抖动在短时间内频繁的加入离开会对集群稳定性有较大影响。tera对于频繁离开加入的节点采取了延迟加入的策略，防止抖动节点对集群的影响。

master这部分逻辑主要在AbnormalNodeMgr内部实现。下面看一下他的主要逻辑。

### 主要的数据结构

```c++
struct NodeAbnormalInfo {
    std::vector<int64_t> deleted_times;     // 节点删除时间组成的向量
    int64_t abnormal_count;                 // 删除过于频繁时，abnormal_count++
    int64_t recovery_time;                  // 节点恢复时间

    NodeAbnormalInfo() : abnormal_count(0), recovery_time(0) {}
  };

std::unordered_map<std::string, NodeAbnormalInfo> nodes_abnormal_infos_;

struct DelayAddNodeInfo {                  // 可以进行恢复的节点
    std::string uuid;                      // 节点uuid
    int64_t recovery_time;                 // 节点恢复时间
  };

std::unordered_map<std::string, DelayAddNodeInfo> delay_add_nodes_;
```

### 主要逻辑

节点删除时会调用abnormal_node_mgr_->RecordNodeDelete

```C++
void AbnormalNodeMgr::RecordNodeDelete(const std::string& addr, const int64_t delete_time) {
  if (FLAGS_abnormal_node_trigger_count < 1) {      // 默认值3
    return;
  }

  MutexLock lock(&mutex_);
  NodeAbnormalInfo& info = nodes_abnormal_infos_[addr];   // 添加到nodes_abnormal_infos_中
  info.deleted_times.emplace_back(delete_time);
  if (info.deleted_times.size() > static_cast<size_t>(FLAGS_abnormal_node_trigger_count)) {
    info.deleted_times.erase(info.deleted_times.begin());
  }

  if (DeleteTooFrequent(info.deleted_times)) {            // 10min内删除3次，认为是抖动
    int64_t last_delete_time = info.deleted_times[info.deleted_times.size() - 1];
    // avoiding overflow, 30 is large enough
    int64_t recovery_wait_time = FLAGS_abnormal_node_auto_recovery_period_s
                                 << (info.abnormal_count > 30 ? 30 : info.abnormal_count);
    if (recovery_wait_time > 24 * 3600) {  // no more than 24 hours
      recovery_wait_time = 24 * 3600;
    }
    info.recovery_time = last_delete_time + recovery_wait_time;    // 最多24小时

    ++info.abnormal_count;
    info.deleted_times.clear();

  }

  // 从恢复队列中删除相应节点
  if (delay_add_nodes_.find(addr) != delay_add_nodes_.end()) {
    delay_add_nodes_.erase(addr);
    abnormal_nodes_count_.Set(delay_add_nodes_.size());
    VLOG(30) << "cancel delay add node, addr: " << addr;
  }
}
```



节点加入之前会调用abnormal_node_mgr_->IsAbnormalNode，如果是非正常接待则直接返回。

```c++
bool AbnormalNodeMgr::IsAbnormalNode(const std::string& addr, const std::string& uuid) {
  MutexLock lock(&mutex_);
  if (nodes_abnormal_infos_.find(addr) == nodes_abnormal_infos_.end()) {
    return false;
  }

  NodeAbnormalInfo& info = nodes_abnormal_infos_[addr];
  
  // 恢复时间已经过去了，还没有恢复就是正常节点了，recovery_time也可能是0，只有DeleteTooFrequent时才有值
  if (get_micros() / 1000000 >= info.recovery_time) {      
    return false;
  } else {
    // 加入延迟恢复队列
    DelayAddNode(addr, uuid);
    return true;
  }
}
```



MasterImpl周期性（60s）调用abnormal_node_mgr_->ConsumeRecoveredNodes将节点从延迟恢复队列中取出节点加入集群。

```c++
void AbnormalNodeMgr::ConsumeRecoveredNodes(std::unordered_map<std::string, std::string>* nodes) {
  MutexLock lock(&mutex_);
  for (const auto& node : delay_add_nodes_) {
    // 达到恢复时间设置
    if (get_micros() / 1000000 >= node.second.recovery_time) {
      nodes->emplace(node.first, node.second.uuid);
    }
  }

  for (const auto& node : *nodes) {
    delay_add_nodes_.erase(node.first);
  }
  abnormal_nodes_count_.Set(delay_add_nodes_.size());
}
```

