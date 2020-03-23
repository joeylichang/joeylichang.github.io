# TabletNode Compact

TabletNode 对tablet进行压缩时，调用的是leveldb的手动压缩接口（启动一个新的线程进行压缩。tera的优化），请求中需要制定lg（既一次压缩一个DB）。如果不指定，则逐个遍历DB进行压缩。各层主要工作如下：

1. RemoteTabletNode::CompactTablet
   1. 将任务加到压缩线程池（默认配置2个线程进行压缩）。
   2. 如果没有SlowdownMode，则等待返回结果在返回response，如果进入SlowdownMode则直接返回response，状态设置为kFlowControlLimited。
2. TabletNodeImpl::CompactTablet：调用TabletIO的Compact手动压缩接口，然后调用TabletIO的GetCompactStatus获取当前的压缩状态并返回。
   1. 注意：Compact调用leveldb的CompactRange是异步执行压缩，但是其内部会调用线程的join接口等待压缩完成之后再返回，会不会很耗时呢？RPC会不会已经超时了呢？
3. TabletIO::Compact：调用LevelDB的CompactRange接口，需要制定lg，在leveldb内部会启动新的线程进行压缩（详细介绍见leveldb部分）。



### RemoteTabletNode::CompactTablet

```c++
void RemoteTabletNode::CompactTablet(google::protobuf::RpcController* controller,
                                     const CompactTabletRequest* request,
                                     CompactTabletResponse* response,
                                     google::protobuf::Closure* done) {
  uint64_t id = request->sequence_id();
  compact_pending_counter.Inc();
  ThreadPool::Task callback =
      std::bind(&RemoteTabletNode::DoCompactTablet, this, controller, request, response, done);
  // Reject all manual compact request when slowdown mode triggered.
  // slowdown模式在quota部分介绍
  if (TsWriteFlowController::Instance().InSlowdownMode()) {
    LOG(WARNING) << "compact fail: " << request->tablet_name()
                 << " due to slowdown write mode triggered.";
    response->set_sequence_id(request->sequence_id());
    response->set_status(kFlowControlLimited);

    done->Run();
    return;
  }
  compact_thread_pool_->AddTask(callback);
}

void RemoteTabletNode::DoCompactTablet(google::protobuf::RpcController* controller,
                                       const CompactTabletRequest* request,
                                       CompactTabletResponse* response,
                                       google::protobuf::Closure* done) {
  uint64_t id = request->sequence_id();
  
  // 主要 统计数据的扣减位置，说明compact_pending_counter统计的是 rpc层等待线程池调用的请求数量
  compact_pending_counter.Dec();
  tabletnode_impl_->CompactTablet(request, response, done);
}
```



### TabletNodeImpl::CompactTablet

```c++
void TabletNodeImpl::CompactTablet(const CompactTabletRequest* request,
                                   CompactTabletResponse* response,
                                   google::protobuf::Closure* done) {
  response->set_sequence_id(request->sequence_id());
  StatusCode status = kTabletNodeOk;
  io::TabletIO* tablet_io = tablet_manager_->GetTablet(request->tablet_name(), request->key_range().key_start(), request->key_range().key_end(), &status);
  if (tablet_io == NULL) {
    LOG(WARNING) << "compact fail to get tablet: " << request->tablet_name() << " ["
                 << DebugString(request->key_range().key_start()) << ", "
                 << DebugString(request->key_range().key_end())
                 << "], status: " << StatusCodeToString(status);
    response->set_status(kKeyNotInRange);
    done->Run();
    return;
  }

  if (request->has_lg_no() && request->lg_no() >= 0) {
    // 调用TabletIO 接口
    tablet_io->Compact(request->lg_no(), &status);
  } else {
    // 表示所有的DB（lg）都需要压缩
    tablet_io->Compact(-1, &status);
  }
  
  CompactStatus compact_status = tablet_io->GetCompactStatus();
  response->set_status(status);
  response->set_compact_status(compact_status);
  uint64_t compact_size = 0;
  tablet_io->GetDataSize(&compact_size);
  response->set_compact_size(compact_size);
  tablet_io->DecRef();
  done->Run();
}
```



### TabletIO::Compact

```c++
bool TabletIO::Compact(int lg_no, StatusCode* status, CompactionType type) {
  {
    MutexLock lock(&mutex_);
    if (status_ != kReady) {
      SetStatusCode(status_, status);
      return false;
    }
    // 状态设置，GetCompactStatus 既返回该变量
    if (compact_status_ == kTableOnCompact) {
      return false;
    }
    compact_status_ = kTableOnCompact;
    db_ref_count_++;
  }
  CHECK_NOTNULL(db_);
  
  // kManualCompaction 是默认参数，内部代码没有使用kMinorCompaction
  // leveldb 的 CompactRange 内部启用新线程执行压缩任务，但是会等待线程结束再返回（join）
  if (type == kManualCompaction) {
    db_->CompactRange(NULL, NULL, lg_no);
  } else if (type == kMinorCompaction) {
    db_->MinorCompact();
  }

  {
    MutexLock lock(&mutex_);
    // 状态设置
    compact_status_ = kTableCompacted;
    db_ref_count_--;
  }
  return true;
}
```

