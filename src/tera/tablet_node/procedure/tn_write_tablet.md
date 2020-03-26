# TN Write Tablet

TabletServer write tablet 主要经过四层的流转，每层主要工作如下：

1. RemoteTabletNode：RPC层的流量控制，权限校验等。

2. TabletNodeImpl

   1. 根据写入的rowkey归属的Tablet，定位写入的TabletIo，每一个TabletIo对应一个WriteTabletTask，每一个WriteTabletTask若干个rowkey。
   2. 遍历所有的TabletIo，调用write接口将相应的rowkeys，写入DB。

3. TabletIO

   1. 校验Tablet的状态。
   2. 调用TabletWriter的写接口完成写操作。

4. TabletWriter

   1. 双buffer机制，批量写入leveldb。
   2. 将所有的请求批量打包成leveldb::WriteBatch （可能有之前的请求，主要看buffer大小）。
   3. 调用TabletIO的WriteBatch接口，其内部调用leveldb::Write将数据写入LevelDB。
   4. 执行完本次操作（一个请求的若干个tablet 写操作中的一个），调用TabletNodeImpl的WriteTabletCallback接口，统计所有tablet的执行情况。

   ##### 注意：TabletWriter 是双buffer机制，待写入leveldb之后会调用TabletNodeImpl层的WriteTabletCallback回调函数，当写入的数据是请求中的数据条目时，会调用RPC的Run方法完成请求，问题是请求的延时增长了，默认是buffer写满和10ms有一个满足就写leveldb，按10ms计算批量写入时有一个5ms的平均延时。

### RemoteTabletNode::WriteTablet

```c++
void RemoteTabletNode::WriteTablet(google::protobuf::RpcController* controller,
                                   const WriteTabletRequest* request, WriteTabletResponse* response,
                                   google::protobuf::Closure* done) {
  int64_t start_micros = get_micros();
  done = WriteDoneWrapper::NewInstance(start_micros, request, response, done);
  static uint32_t last_print = time(NULL);
  int32_t row_num = request->row_list_size();
  write_request_counter.Add(row_num);
  
  // 写请求在RPC层总限流数，10w
  if (write_pending_counter.Get() > FLAGS_tera_request_pending_limit) {
    // 回填response
  } else {
    // check user identification & access
    if (!access_entry_->VerifyAndAuthorize(request, response)) {
      // 回填response
      return;
    }

    // sum write bytes
    int64_t sum_write_bytes = 0;
    for (int32_t row_index = 0; row_index < row_num; ++row_index) {
      sum_write_bytes += request->row_list(row_index).ByteSize();
    }
    // 流控逻辑，后续介绍
    if (!TsWriteFlowController::Instance().TryWrite(sum_write_bytes)) {
      // 回填response
      return;
    }
    
    // 写请求数量如果超过限制流量一定比例需要check quota进行流控 ， 比例0.1 既 1w
    if (write_pending_counter.Get() >=
        FLAGS_tera_request_pending_limit * FLAGS_tera_quota_unlimited_pending_ratio) {
      if (!quota_entry_->CheckAndConsume(
              request->tablet_name(),
              quota::OpTypeAmountList{std::make_pair(kQuotaWriteReqs, row_num),
                                      std::make_pair(kQuotaWriteBytes,       sum_write_bytes)})) {
        // 回填response
        return;
      }
    }
    write_pending_counter.Add(row_num);
    
    // timer 在内部并没有超时 的机制，只是在主循环中统计有用到，后面介绍
    WriteRpcTimer* timer = new WriteRpcTimer(request, response, done, start_micros);
    RpcTimerList::Instance()->Push(timer);
    // 绑定回调DoWriteTablet，内部除了统计数据（后续统一梳理统计数据），直接调用TabletNodeImpl接口
    ThreadPool::Task callback = std::bind(&RemoteTabletNode::DoWriteTablet, this, controller, request, response, done, timer);
    write_thread_pool_->AddTask(callback);
  }
}
```



### TabletNodeImpl::WriteTablet

```c++
struct WriteTabletTask {
    std::vector<const RowMutationSequence*> row_mutation_vec;
    std::vector<StatusCode> row_status_vec;
    std::vector<int32_t> row_index_vec;
    std::shared_ptr<Counter> row_done_counter;

    const WriteTabletRequest* request;
    WriteTabletResponse* response;
    google::protobuf::Closure* done;
    WriteRpcTimer* timer;

    WriteTabletTask(const WriteTabletRequest* req, WriteTabletResponse* resp,
                    google::protobuf::Closure* d, WriteRpcTimer* t, std::shared_ptr<Counter> c)
        : row_done_counter(c), request(req), response(resp), done(d), timer(t) {}
  };

void TabletNodeImpl::WriteTablet(const WriteTabletRequest* request, WriteTabletResponse* response,
                                 google::protobuf::Closure* done, WriteRpcTimer* timer) {
  response->set_sequence_id(request->sequence_id());
  StatusCode status = kTabletNodeOk;

  std::map<io::TabletIO*, WriteTabletTask*> tablet_task_map;
  std::map<io::TabletIO*, WriteTabletTask*>::iterator it;

  int32_t row_num = request->row_list_size();
  if (row_num == 0) {
    // 回填response，删除timer
    return;
  }

  std::shared_ptr<Counter> row_done_counter(new Counter);
  for (int32_t i = 0; i < row_num; i++) {
    io::TabletIO* tablet_io =
        tablet_manager_->GetTablet(request->tablet_name(), request->row_list(i).row_key(), &status);
    if (tablet_io == NULL) {
      write_range_error_counter.Inc();
    }
    it = tablet_task_map.find(tablet_io);
    WriteTabletTask* tablet_task = NULL;
    if (it == tablet_task_map.end()) {
      // keep one ref to tablet_io，根据tablet 分配rowkey
      tablet_task = tablet_task_map[tablet_io] =
          new WriteTabletTask(request, response, done, timer, row_done_counter);
    } else {
      if (tablet_io != NULL) {
        tablet_io->DecRef();
      }
      tablet_task = it->second;
    }
    // row_list是写入的数据，row_status_vec返回的状态
    tablet_task->row_mutation_vec.push_back(&request->row_list(i));
    tablet_task->row_status_vec.push_back(kTabletNodeOk);
    tablet_task->row_index_vec.push_back(i);
  }

  // reserve response status list space
  response->set_status(kTabletNodeOk);
  response->mutable_row_status_list()->Reserve(row_num);
  for (int32_t i = 0; i < row_num; i++) {
    response->mutable_row_status_list()->AddAlreadyReserved();
  }

  for (it = tablet_task_map.begin(); it != tablet_task_map.end(); ++it) {
    io::TabletIO* tablet_io = it->first;
    WriteTabletTask* tablet_task = it->second;
    if (tablet_io == NULL) {
      // 更新统计数据，和回填tablet_task->row_status_vec，然后调用请求结束的回调，后面介绍
      WriteTabletFail(tablet_task, kKeyNotInRange);
      
      // 调用tabletIo的写操作
    } else if (!tablet_io->Write(
                   &tablet_task->row_mutation_vec, &tablet_task->row_status_vec,
                   request->is_instant(),
                   std::bind(&TabletNodeImpl::WriteTabletCallback, this, tablet_task, _1, _2),
                   &status)) {
      tablet_io->DecRef();
      WriteTabletFail(tablet_task, status);
    } else {
      tablet_io->DecRef();
    }
  }
}
```



### TabletIO::Write

```c++
bool TabletIO::Write(std::vector<const RowMutationSequence*>* row_mutation_vec,
                     std::vector<StatusCode>* status_vec, bool is_instant, WriteCallback callback,
                     StatusCode* status) {
  {
    MutexLock lock(&mutex_);
    // 状态校验
    // IsUrgentUnload 标准是 unload是否重复执行3次以上
    if ((status_ != kReady && status_ != kUnloading) || IsUrgentUnload()) {
      if (status_ == kUnloading2) {
        // keep compatable for old sdk protocol
        // we can remove this in the future.
        SetStatusCode(kUnloading, status);
      } else {
        SetStatusCode(status_, status);
      }
      return false;
    }
    db_ref_count_++;
  }
  
  // 调用tabletwriter的接口，内部是双buffer，写入内存之后待批量刷leveldb
  bool ret = async_writer_->Write(row_mutation_vec, status_vec, is_instant, callback, status);
  if (!ret) {
    counter_.write_reject_rows.Add(row_mutation_vec->size());
  }

  {
    MutexLock lock(&mutex_);
    db_ref_count_--;
  }
  return ret;
}
```



### TabletWriter::Write

```c++
bool TabletWriter::Write(std::vector<const RowMutationSequence*>* row_mutation_vec,
                         std::vector<StatusCode>* status_vec, bool is_instant,
                         WriteCallback callback, StatusCode* status) {
  static uint32_t last_print = time(NULL);
  const uint64_t MAX_PENDING_SIZE = FLAGS_tera_asyncwriter_pending_limit * 1024UL;

  MutexLock lock(&task_mutex_);
  if (stopped_) {
    LOG(ERROR) << "tablet writer is stopped";
    SetStatusCode(kAsyncNotRunning, status);
    return false;
  }
  if (active_buffer_size_ >= MAX_PENDING_SIZE || tablet_busy_) {
    uint32_t now_time = time(NULL);
    if (now_time > last_print) {
      last_print = now_time;
    }
    SetStatusCode(kTabletNodeIsBusy, status);
    return false;
  }

  uint64_t request_size = CountRequestSize(*row_mutation_vec, tablet_->KvOnly());
  WriteTask task;
  task.row_mutation_vec = row_mutation_vec;
  task.status_vec = status_vec;
  task.callback = callback;

  // 加入active_buffer_，默认配合10ms 切换一次buffer，然后刷盘，写leveldb
  active_buffer_->push_back(task);
  active_buffer_size_ += request_size;
  active_buffer_instant_ |= is_instant;
  
  // 如果buffer超过设置的闸值（1M）或者请求中设置了is_instant 立即刷盘参数，则立即切换buffer 刷盘
  if (active_buffer_size_ >= FLAGS_tera_asyncwriter_sync_size_threshold * 1024UL ||
      active_buffer_instant_) {
    write_event_.Set();
  }
  return true;
}
```

下面看一下写入leveldb的逻辑

```c++
StatusCode TabletWriter::FlushToDiskBatch(WriteTaskBuffer* task_buffer) {
  int64_t start_ts, check_cost, batch_cost, write_cost, finish_cost;

  start_ts = get_micros();
  CheckRows(task_buffer);
  check_cost = get_micros();

  leveldb::WriteBatch batch;
  
  // 根据buffer中的task信息，组装成batch请求
  // 内部会根据rowkey的格式（KV、删除faily等）进行封装，然后putkey、val到batch中
  BatchRequest(task_buffer, &batch);
  batch_cost = get_micros();
  StatusCode status = kTabletNodeOk;
  if (tablet_->IsUrgentUnload()) {
    // 直接不写
    LOG(INFO) << "tablet unload slow, reject to write log and memtable";
  } else {
    const bool disable_wal = false;
    // 调用tablet的接口，直接写level
    // 默认tera_sync_log 为true，既将log的file cache写入磁盘
    tablet_->WriteBatch(&batch, disable_wal, FLAGS_tera_sync_log, &status);
  }
  batch.Clear();
  write_cost = get_micros();

  // 内部更新统计信息和写请求返回的状态，调用task的回调既TableImpl的WriteTabletCallback
  FinishTask(task_buffer, status);
  finish_cost = get_micros();
  int64_t check_delay = check_cost - start_ts;
  int64_t batch_delay = batch_cost - check_cost;
  int64_t write_delay = write_cost - batch_cost;
  int64_t finish_delay = finish_cost - write_cost;

  flush_to_disk_check_delay.Add(check_delay);
  flush_to_disk_batch_delay.Add(batch_delay);
  flush_to_disk_write_delay.Add(write_delay);
  flush_to_disk_finish_delay.Add(finish_delay);

  VLOG(7) << "finish a batch: " << task_buffer->size()
          << ", cost(check/batch/write/finish): " << check_delay << "/" << batch_delay << "/"
          << write_delay << "/" << finish_delay;
  return status;
}
```

```c++
bool TabletIO::WriteBatch(leveldb::WriteBatch* batch, bool disable_wal, bool sync,
                          StatusCode* status) {
  leveldb::WriteOptions options;
  options.disable_wal = disable_wal;
  options.sync = sync;

  CHECK_NOTNULL(db_);

  // 核心：调用leveldb写入数据，leveldb部分介绍
  leveldb::Status db_status = db_->Write(options, batch);
  if (!db_status.ok()) {
    LOG(ERROR) << "fail to batch write to tablet: " << tablet_path_ << ", " << db_status.ToString();
    SetStatusCode(kIOError, status);
    return false;
  }
  counter_.write_size.Add(batch->DataSize());
  row_write_bytes.Add(batch->DataSize());
  SetStatusCode(kTabletNodeOk, status);
  return true;
}
```

最后看一下传入tabletIo的回调逻辑：

```c++
void TabletNodeImpl::WriteTabletCallback(WriteTabletTask* tablet_task,
                                         std::vector<const RowMutationSequence*>* row_mutation_vec,
                                         std::vector<StatusCode>* status_vec) {
  
  // TabletNodeImpl 根据 tablet 分成了若干个task，这里只返回了一个task
  int32_t index_num = tablet_task->row_index_vec.size();
  for (int32_t i = 0; i < index_num; i++) {
    int32_t index = tablet_task->row_index_vec[i];
    tablet_task->response->mutable_row_status_list()->Set(index, (*status_vec)[i]);
  }

  if (tablet_task->row_done_counter->Add(index_num) == tablet_task->request->row_list_size()) {
    
    // 此处才真正的结束一个请求，期间等待了一个批量写入的等待时间（10ms或者buffer写满）
    tablet_task->done->Run();
    if (NULL != tablet_task->timer) {
      RpcTimerList::Instance()->Erase(tablet_task->timer);
      delete tablet_task->timer;
    }
  }

  delete tablet_task;
}
```

##### 注意

1. 此时tablet_task->response->mutable_row_status_list()->Set(index, (*status_vec)[i]) 回填的是leveldb层真实的是否写入成功。
2. 如果write请求中有设置is_instant参数，则会立即刷盘，是时间换吞吐。
3. 总结：
   1. 写入leveldb可以用吞吐 和 延时的balance，因为写入的是DFS与本地leveldb的追加写还是有所区别的。
   2. 网络交互的延迟会更高一些如果不增加每次写入，吞吐势必有影响。
   3. 反之，如果做了批量处理，吞吐会上来，但是平均延时会增加。