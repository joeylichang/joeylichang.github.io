# Master Unload Tablet 流程

Master侧unload tablet的流程中状态的转换不是内部自定的，与tablet的状态相关。

### Tablet状态初始化

```c++
UnloadTabletProcedure::UnloadTabletProcedure(TabletPtr tablet, ThreadPool* thread_pool,
                                             bool is_sub_proc)
    : id_(std::string("UnloadTablet:") + tablet->GetPath() + ":" + TimeStamp()),
      tablet_(tablet),
      unload_retrys_(0),
      unload_request_dispatching_(false),
      kick_ts_succ_(true),
      done_(false),
      ts_unload_busying_(false),
      is_sub_proc_(is_sub_proc),
      thread_pool_(thread_pool) {
  PROC_LOG(INFO) << "unload tablet begin, tablet: " << tablet_;
  // I played a trick here by setting tablet status to kTableReady when user
  // want to unload a
  // tablet currently in status kTableUnloading
  if (tablet_->GetStatus() == TabletMeta::kTabletUnloading ||
      tablet_->GetStatus() == TabletMeta::kTabletUnloadFail) {
    tablet_->SetStatus(TabletMeta::kTabletReady);
  }
  if (tablet_->GetStatus() == TabletMeta::kTabletDelayOffline) {
    tablet_->DoStateTransition(TabletEvent::kTsOffline);
  }
}
```

tablet的初始化状态只能是kTsOffline、kTabletReady，kTabletUnloading 和 kTabletUnloadFail 转换为kTabletReady，kTabletDelayOffline转化为kTsOffline。其他状态则直接结束。



### Tablet状态转换逻辑

```c++
TabletEvent UnloadTabletProcedure::GenerateEvent() {
  // offline 或者 unloadfail 则直接结束，从初始化开始
  // 说明已经执行一遍unload的流程了，仍然失败则结束
  if (tablet_->GetStatus() == TabletMeta::kTabletOffline ||
      tablet_->GetStatus() == TabletMeta::kTabletUnloadFail) {
    return TabletEvent::kEofEvent;
  }
  
  // 节点down，则直接，则任务unload成功，回调中值简单的对unload计数 -1
  if (tablet_->GetTabletNode()->NodeDown()) {
    return TabletEvent::kTsOffline;
  }
  
  // 初始是ready状态
  if (tablet_->GetStatus() == TabletMeta::kTabletReady) {
    if (!tablet_->GetTabletNode()->CanUnload()) {
      // kTsUnloadBusy 回调直接返回，等待unload数目变成设定闸值之下继续，默认单节点50个
      return TabletEvent::kTsUnloadBusy;
    }
    // 指定unload逻辑
    return TabletEvent::kUnLoadTablet;
  }

  if (tablet_->GetStatus() == TabletMeta::kTabletUnloading) {
    // 表示unload请求已经返送，但未返回
    if (unload_request_dispatching_) {
      // 回调直接返回，等待回调结束
      return TabletEvent::kWaitRpcResponse;
    }
    // 默认5次
    if (unload_retrys_ <= FLAGS_tera_master_impl_retry_times) {
      // 回调中对ts的unload计数-1
      return TabletEvent::kTsUnLoadSucc;
    }
    // if exhausted all unload retries, wait kick result
    if (kick_ts_succ_) {
      // kick 成功则，认为unload成功，所以等结果返回即可
      return TabletEvent::kWaitRpcResponse;
    }
    // kTabletUnloading状态且，请求返回看，说明unloadfail，回调中对ts的unload计数+1
    return TabletEvent::kTsUnLoadFail;
  }
  return TabletEvent::kEofEvent;
}
```

