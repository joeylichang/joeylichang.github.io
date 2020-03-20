# TabletNode Scan

TabletServer write tablet 主要经过三层的流转，每层主要工作如下：

1. RemoteTabletNode
   1. RPC层的流控，默认是1000。
   2. 权限认证。
   3. 将请求交给scan_rpc_schedule_去调度（详见[RPC调度](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/rpc/tn_rpc.md)介绍）
2. TabletNodeImpl：获取TabletIo，调用其ScanRows进行遍历。
3. TabletIO：将Scan请求分成了三种。
   1. BatchScan：既一次遍历需要多个RPC请求，期间需要缓存中间状态（详见[BatchScan](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/procedure/tn_batch_scan.md)介绍）。
   2. KV非BatchScan（本部分重点介绍）。
   3. 非KV非BatchScan（本部分重点介绍）。

### RemoteTabletNode::ScanTablet

```c++
void RemoteTabletNode::ScanTablet(google::protobuf::RpcController* controller,
                                  const ScanTabletRequest* request, ScanTabletResponse* response, google::protobuf::Closure* done) {
  done = ScanDoneWrapper::NewInstance(get_micros(), request, response, done, quota_entry_);
  VLOG(8) << "accept RPC (ScanTablet): [" << request->table_name() << "] "
          << tera::utils::GetRemoteAddress(controller);
  scan_request_counter.Inc();
  
  // RPC层流控
  if (scan_pending_counter.Get() > FLAGS_tera_scan_request_pending_limit) {
    response->set_sequence_id(request->sequence_id());
    response->set_status(kTabletNodeIsBusy);
    scan_reject_counter.Inc();
    done->Run();
    VLOG(8) << "finish RPC (ScanTablet)";
  } else {
    // check user identification & access
    // 权限校验
    if (!access_entry_->VerifyAndAuthorize(request, response)) {
      response->set_sequence_id(request->sequence_id());
      VLOG(20) << "Access VerifyAndAuthorize failed for ScanTablet";
      done->Run();
      return;
    }
    // rpctask 用于scan_rpc_schedule_调度
    ScanRpc* rpc = new ScanRpc(controller, request, response, done);
    if (scan_pending_counter.Get() >=
        FLAGS_tera_request_pending_limit * FLAGS_tera_quota_unlimited_pending_ratio) {
      if (!DoQuotaScanRpcRetry(rpc)) {
        VLOG(8) << "ScanTablet Rpc push to QuotaRetry queue";
        return;
      }
    }
    scan_pending_counter.Inc();
    scan_rpc_schedule_->EnqueueRpc(request->table_name(), rpc);
    
    // DoScheduleRpc内部调用TabletNodeImpl接口，并对scan_pending_counter进行-1
    // 所以，scan_pending_counter只是RPC层的控制，如果是异步则实际请求并未完成，scan、read是同步，write是异步
    scan_thread_pool_->AddTask(
        std::bind(&RemoteTabletNode::DoScheduleRpc, this, scan_rpc_schedule_.get()));
  }
}
```



### TabletNodeImpl::ScanTablet

```c++
void TabletNodeImpl::ScanTablet(const ScanTabletRequest* request, ScanTabletResponse* response, google::protobuf::Closure* done) {
  const int64_t PACK_MAX_SIZE = static_cast<int64_t>(FLAGS_tera_tabletnode_scan_pack_max_size)
                                << 10;
  // const std::string& start_key = request->key_range().key_start();
  // const std::string& end_key = request->key_range().key_end();
  // 为使用
  int64_t buffer_limit = request->buffer_limit();
  if (buffer_limit > PACK_MAX_SIZE) {
    buffer_limit = PACK_MAX_SIZE;
  }
  // VLOG(5) << "ScanTablet() start=[" << start_key
  //    << "], end=[" << end_key << "]";
  if (request->has_sequence_id()) {
    response->set_sequence_id(request->sequence_id());
  }

  StatusCode status = kTabletNodeOk;
  io::TabletIO* tablet_io = NULL;
  tablet_io = tablet_manager_->GetTablet(request->table_name(), request->start(), &status);

  if (tablet_io == NULL) {
    scan_range_error_counter.Inc();
    response->set_status(status);
    done->Run();
  } else {
    response->set_end(tablet_io->GetEndKey());
    // 核心逻辑
    if (!tablet_io->ScanRows(request, response, done)) {
      scan_error_counter.Inc();
    }
    tablet_io->DecRef();
  }
}
```



### TabletIO::ScanRows

```c++
bool TabletIO::ScanRows(const ScanTabletRequest* request, ScanTabletResponse* response,
                        google::protobuf::Closure* done) {
  StatusCode status = kTabletNodeOk;
  {
    // 状态检查
    MutexLock lock(&mutex_);
    if ((status_ != kReady && status_ != kUnloading) || IsUrgentUnload()) {
      if (status_ == kUnloading2) {
        // keep compatable for old sdk protocol
        // we can remove this in the future.
        SetStatusCode(kUnloading, &status);
      } else {
        SetStatusCode(status_, &status);
      }
      response->set_status(status);
      done->Run();
      return false;
    }
    db_ref_count_++;
  }

  bool success = false;
  // slide window of batchscan use unique rpc session
  // so, has_session_id means batchscan
  // 区分是不是BatchScan通过session_id，如果有且大于0，则表示BatchScan，所有的自请求都有想用的session_id
  // kv 非BatchSan
  if (kv_only_ && !request->has_session_id()) {
    success = ScanKvsRestricted(request, response, done);
  } else if (request->has_session_id() && request->session_id() > 0) {
    batch_scan_count.Inc();
    // BatchSan 内部也会区分是都是kv，见其他文章介绍
    success = HandleScan(request, response, done);
  } else {
    sync_scan_count.Inc();
    // readyable binary的非BatchSan
    success = ScanRowsRestricted(request, response, done);
  }
  {
    MutexLock lock(&mutex_);
    db_ref_count_--;
  }
  return success;
}
```



##### TabletIO::ScanRowsRestricted

ScanRowsRestricted 内部调用的LowLevelScan

```C++
bool TabletIO::ScanRowsRestricted(const ScanTabletRequest* request, ScanTabletResponse* response, google::protobuf::Closure* done) {
  std::string start_tera_key;
  std::string end_row_key;
  // 根据key类型进行编码start_key 和 end_key
  SetupScanKey(request, &start_tera_key, &end_row_key);

  // 遍历的参数设置，详见BatchSacn介绍，逻辑相同
  ScanOptions scan_options;
  SetupScanRowOptions(request, &scan_options);

  bool complete = false;

  StatusCode status = kTabletNodeOk;
  bool ret = false;

  int64_t start_scan_us = get_micros();

  if (LowLevelScan(start_tera_key, end_row_key, scan_options, response->mutable_results(), response->mutable_next_start_point(), &read_row_count, &read_cell_count, &read_bytes, &complete, &status)) {
    response->set_complete(complete);
    ret = true;
  }

  response->set_data_size(read_bytes);
  response->set_row_count(read_row_count);
  response->set_cell_count(read_cell_count);

  response->set_status(status);
  done->Run();
  return ret;
}

bool TabletIO::LowLevelScan(const std::string& start_tera_key, const std::string& end_row_key, const ScanOptions& scan_options, RowResult* values, KeyValuePair* next_start_point, uint32_t* read_row_count, uint32_t* read_cell_count, uint32_t* read_bytes, bool* complete, StatusCode* status) {
  leveldb::Iterator* it = NULL;
  // 迭代器初始化，详见BatchSacn介绍，逻辑相同
  StatusCode ret_code = InitScanIterator(start_tera_key, end_row_key, scan_options, &it);
  if (ret_code != kTabletNodeOk) {
    SetStatusCode(ret_code, status);
    return false;
  }

  // 遍历的上下文，目的是调用底层LowLevelScan，复用BatchSacn逻辑
  ScanContext* context = new ScanContext;
  context->compact_strategy = ldb_options_.compact_strategy_factory->NewInstance();
  context->version_num = 1;
  context->qu_num = 1;
  // 详见BatchSacn介绍，逻辑相同
  bool ret =
      LowLevelScan(start_tera_key, end_row_key, scan_options, it, context, values, next_start_point,
                   read_row_count, read_cell_count, read_bytes, complete, status);
  delete it;
  delete context->compact_strategy;
  delete context;
  return ret;
}
```



##### TabletIO::ScanKvsRestricted

```c++
bool TabletIO::ScanKvsRestricted(const ScanTabletRequest* request, ScanTabletResponse* response, google::protobuf::Closure* done) {
  bool ret = false;
  // leveldb的参数设置
  ScanOption scan_option;
  scan_option.set_snapshot_id(request->snapshot_id());
  scan_option.mutable_key_range()->set_key_start(request->start());
  scan_option.mutable_key_range()->set_key_end(request->end());
  if (request->has_buffer_limit()) {
    scan_option.set_size_limit(request->buffer_limit());
  } else {
    scan_option.set_size_limit(FLAGS_tera_tabletnode_scan_pack_max_size << 10);
  }
  scan_option.set_round_down(request->round_down());
  bool complete = false;

  StatusCode status = kTabletNodeOk;
  if (Scan(scan_option, response->mutable_results()->mutable_key_values(), &read_row_count, &read_bytes, &complete, &status)) {
    response->set_complete(complete);
    ret = true;
  }
  
  response->set_data_size(read_bytes);
  response->set_row_count(read_row_count);
  response->set_cell_count(read_row_count);

  response->set_status(status);
  done->Run();
  return ret;
}
```

下面看一下核心逻辑Scan：

```c++
bool TabletIO::Scan(const ScanOption& option, KeyValueList* kv_list, uint32_t* read_row_count, uint32_t* read_bytes, bool* complete, StatusCode* status) {
  std::string start = option.key_range().key_start();
  std::string end = option.key_range().key_end();\
  // start_key_ / end_key_ 是tablet的key范围
  if (start < start_key_) {
    start = start_key_;
  }
  if (end.empty() || (!end_key_.empty() && end > end_key_)) {
    end = end_key_;
  }

  // TTL-KV : key_operator_::Compare会解RawKey([row_key | expire_timestamp])
  // 因此传递给Leveldb的Key一定要保证以expire_timestamp结尾.
  // start_key / end_key 的编码（时间戳、类型等内容）
  std::unique_ptr<leveldb::CompactStrategy> strategy(nullptr);
  if (RawKeyType() == TTLKv) {
    if (!start.empty()) {
      std::string start_key;
      key_operator_->EncodeTeraKey(start, "", "", 0, leveldb::TKT_FORSEEK, &start_key);
      start.swap(start_key);
    }
    if (!end.empty()) {
      std::string end_key;
      key_operator_->EncodeTeraKey(end, "", "", 0, leveldb::TKT_FORSEEK, &end_key);
      end.swap(end_key);
    }
    strategy.reset(ldb_options_.compact_strategy_factory->NewInstance());
  }

  *read_row_count = 0;
  *read_bytes = 0;

  uint64_t snapshot_id = option.snapshot_id();
  leveldb::ReadOptions read_option(&ldb_options_);
  read_option.verify_checksums = FLAGS_tera_leveldb_verify_checksums;
  
  // 用户的snapshot_id 转换为 leveldb的snapshot_id，有一个map进行映射
  if (snapshot_id != 0 && !SnapshotIDToSeq(snapshot_id, &read_option.snapshot)) {
    *status = kSnapshotNotExist;
    return false;
  }
  read_option.rollbacks = rollbacks_;

  // ReadOptions 目的是获取迭代器进行遍历
  std::unique_ptr<leveldb::Iterator> it(db_->NewIterator(read_option));
  if (it->status().IsShutdownInProgress()) {
    *status = kKeyNotInRange;
    return false;
  }

  it->Seek(start);
  // round down is just for internal meta scan
  if (option.round_down()) {
    if (it->Valid() && key_operator_->Compare(it->key(), start) > 0) {
      it->Prev();
      if (!it->Valid()) {
        it->SeekToFirst();
      }
    } else if (!it->Valid()) {
      it->SeekToLast();
    }
  }

  int64_t pack_size = 0;
  for (; it->Valid(); it->Next()) {
    leveldb::Slice key = it->key();
    leveldb::Slice value = it->value();
    *read_bytes += it->key().size() + it->value().size();
    *read_row_count += 1;
    
    // 表示遍历超过end_key 完成所有的遍历了
    if (RawKeyType() == TTLKv) {  // only compare row key
      *complete = (!end.empty() && key_operator_->Compare(key, end) >= 0);
    } else {
      *complete = (!end.empty() && key.compare(end) >= 0);
    }

    // 判断结束条件： 遍历到最后一个key、请求大小达到限制（用户指定）
    if (*complete || (option.size_limit() > 0 && pack_size > option.size_limit())) {
      break;
    }

    // 主要是判断是否过期
    if (strategy && strategy->ScanDrop(key, 0)) {
      VLOG(10) << "[KV-Scan] key:[" << key.ToString() << "] Dropped.";
    } else {
      // 结果回填到respones的result中
      KeyValuePair* pair = kv_list->Add();
      if (RawKeyType() == TTLKv) {
        pair->set_key(key.data(), key.size() - sizeof(int64_t));
      } else {
        pair->set_key(key.data(), key.size());
      }
      pair->set_value(value.data(), value.size());
      pack_size += pair->key().size() + pair->value().size();
    }

    if (db_->IsShutdown1Finished()) {
      // return early on waiting_for_shutdown2_, igrone rows haven't scan
      TABLET_UNLOAD_LOG << "break scan kv before iterator next";
      *status = kKeyNotInRange;
      return false;
    }
  }
  if (!it->Valid()) {
    *complete = true;
  }

  return true;
}
```

