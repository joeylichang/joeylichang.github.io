# TabletNode Unload tablet

### RemoteTabletNode::UnloadTablet

```c++
void RemoteTabletNode::UnloadTablet(google::protobuf::RpcController* controller,
                                    const UnloadTabletRequest* request,
                                    UnloadTabletResponse* response,
                                    google::protobuf::Closure* done) {
  uint64_t id = request->sequence_id();
  response->set_sequence_id(id);

  // rpc层对卸载的tablet做了一层缓存，对于已经在卸载map的直接查询tablet状态并返回
  if (request->has_path()) {
    std::lock_guard<std::mutex> lock(tablets_ctrl_mutex_);
    const std::string& tablet_path = request->path();
    if (tablets_ctrl_status_.find(tablet_path) != tablets_ctrl_status_.end()) {
      ThreadPool::Task query_task = std::bind(&RemoteTabletNode::DoQueryTabletUnloadStatus, this,
                                              controller, request, response, done);
      lightweight_ctrl_thread_pool_->AddTask(query_task);
      return;
    }
  }

  // 默认20个线程，超过20个并发度直接返回
  if (ctrl_thread_pool_->PendingNum() > FLAGS_tera_tabletnode_ctrl_thread_num) {
    response->set_status(kTabletNodeIsBusy);
    done->Run();
    return;
  }
  if (request->has_path()) {
    tablets_ctrl_status_[request->path()] = TabletCtrlStatus::kCtrlWaitUnload;
  }

  ThreadPool::Task callback =
      std::bind(&RemoteTabletNode::DoUnloadTablet, this, controller, request, response, done);
  ctrl_thread_pool_->AddTask(callback);
}

void RemoteTabletNode::DoUnloadTablet(google::protobuf::RpcController* controller,
                                      const UnloadTabletRequest* request,
                                      UnloadTabletResponse* response,
                                      google::protobuf::Closure* done) {
  uint64_t id = request->sequence_id();

  std::string tablet_path;
  if (request->has_path()) {
    tablet_path = request->path();
    std::lock_guard<std::mutex> lock(tablets_ctrl_mutex_);
    // 更新状态，注意这只是RPC的状态，表示RPC已经开始响应这个请求了，并不是底层的状态
    tablets_ctrl_status_[tablet_path] = TabletCtrlStatus::kCtrlUnloading;
  }

  tabletnode_impl_->UnloadTablet(request, response);
  {
    std::lock_guard<std::mutex> lock(tablets_ctrl_mutex_);
    tablets_ctrl_status_.erase(tablet_path);
  }
  LOG(INFO) << "finish RPC (UnloadTablet) id: " << id;
  done->Run();
}
```



### TabletNodeImpl::UnloadTablet

```c++
bool TabletNodeImpl::UnloadTablet(const std::string& tablet_name, const std::string& start,
                                  const std::string& end, StatusCode* status) {
  io::TabletIO* tablet_io = tablet_manager_->GetTablet(tablet_name, start, end, status);
  if (tablet_io == NULL) {
    StatusCodeToString(*status);
    *status = kKeyNotInRange;
    return false;
  }

  // 核心逻辑，内部分两阶段进行
  if (!tablet_io->Unload(status)) {
    io::TabletIO::TabletStatus tablet_status = tablet_io->GetStatus();
    if (tablet_status == io::TabletIO::TabletStatus::kUnloading ||
        tablet_status == io::TabletIO::TabletStatus::kUnloading2) {
    } else {
    }
    *status = (StatusCode)tablet_status;
    tablet_io->DecRef();
    return false;
  }
  tablet_io->DecRef();

  if (!tablet_manager_->RemoveTablet(tablet_name, start, end, status)) {
  }
  *status = kTabletNodeOk;
  return true;
}
```



### TabletIO::Unload

```c++
bool TabletIO::Unload(StatusCode* status) {
  {
    MutexLock lock(&mutex_);
    // inc try unload times
    // 用户urgentunload 标准的判定，
    ++try_unload_count_;
    LOG(INFO) << "tablet " << tablet_path_ << " unload try times:" << try_unload_count_;
    if (status_ != kReady) {
      SetStatusCode(status_, status);
      return false;
    }
    // 状态设定
    status_ = kUnloading;
    db_ref_count_++;
  }

  /*
   * 1. 遍历所有的lg（DBIpml），调用Shutdown1
   *     1.1 等待所有的压缩任务完成
   *     1.2 dump内存数据
   * 2. gc处理
   */
  leveldb::Status s = db_->Shutdown1();
  {
    MutexLock lock(&mutex_);
    status_ = kUnloading2;
  }

  uint32_t retry = 0;
  // 等待引用计数清0，否则还会有来自TabletIO的请求
  while (db_ref_count_ > 1) {
    ThisThread::Sleep(FLAGS_tera_io_retry_period);
  }

  // 异步写操作的类
  async_writer_->Stop();
  delete async_writer_;
  async_writer_ = NULL;

  if (s.ok()) {
    /*
     * 1. 遍历所有的lg（DBIpml），Shutdown2
     *     1.1 dump 内存数据
     * 2. 停止日志，日志阶段，删除多余日志文件
     *
     * 分两阶段进行Shutdown，主要是因为第一阶段在进行必要处理之后，还会等待写操作完成，此时内存很坑还有数据，优雅退出的策略
     */
    db_->Shutdown2();
  } else {
    LOG(INFO) << "[Unload] shutdown1 failed, keep log " << tablet_path_;
  }

  delete scan_context_manager_;
  delete db_;
  db_ = NULL;

  delete ldb_options_.filter_policy;
  // 成员变量ldb_options_ 的清理
  TearDownOptionsForLG();

  {
    MutexLock lock(&mutex_);
    status_ = kNotInit;
    db_ref_count_--;
  }
  return true;
}
```

