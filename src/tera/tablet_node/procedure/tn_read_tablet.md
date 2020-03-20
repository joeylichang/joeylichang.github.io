# TN Write Tablet

TabletServer write tablet 主要经过三层的流转，每层主要工作如下：

1. RemoteTabletNode::ReadTablet
   1. RPC层流量控制，默认读写供10w（注意：转发下层之后，计数会+1，既只是RPC流控）。
   2. 权限和quota的校验。
   3. 交给read_rpc_schedule_调度（详见[RPC调度](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/rpc/tn_rpc.md)介绍），读请求的线程池是40。
   4. 调用TabletNodeImpl的ReadTablet接口。
2. TabletNodeImpl::ReadTablet
   1. TabletNodeImpl层会将一个读请求映射为一个ReadTabletTask。
   2. 调用ReadTabletTask的StartRead拆分成多个ShardRequest，每个ShardRequest调用ReadTabletTask的DoRead（内部调用TabletIo的ReadCells接口）进行读操作，每个DoRead结束会调用FinishShardRequest内部会收集读的结果，全部完成时返回RPC请求。
3. TabletIO::ReadCells：对读请求分三种情况
   1. kv读：调用leveldb Get接口。
   2. 指定col qua条件的读，调用LowLevelSeek，通过迭代器对读到的数据进行过滤。
   3. 未指定col qua条件，既读一整行，调用LowLevelScan，复用Scan的逻辑。

### RemoteTabletNode::ReadTablet

```c++
void RemoteTabletNode::ReadTablet(google::protobuf::RpcController* controller, const ReadTabletRequest* request, ReadTabletResponse* response, google::protobuf::Closure* done) {
  int64_t start_micros = get_micros();
  done = ReadDoneWrapper::NewInstance(start_micros, request, response, done, quota_entry_);

  static uint32_t last_print = time(NULL);
  int32_t row_num = request->row_info_list_size();
  read_request_counter.Add(row_num);
  
  // 流控 10w
  if (read_pending_counter.Get() > FLAGS_tera_request_pending_limit) {
    response->set_sequence_id(request->sequence_id());
    response->set_status(kTabletNodeIsBusy);
    read_reject_counter.Add(row_num);
    done->Run();
    uint32_t now_time = time(NULL);
    if (now_time > last_print) {
      LOG(WARNING) << "Too many pending read requests, return TabletNode Is Busy!";
      last_print = now_time;
    }
    VLOG(8) << "finish RPC (ReadTablet)";
  } else {
    // check user identification & access
    if (!access_entry_->VerifyAndAuthorize(request, response)) {
      response->set_sequence_id(request->sequence_id());
      VLOG(20) << "Access VerifyAndAuthorize failed for ReadTablet";
      done->Run();
      return;
    }
    if (read_pending_counter.Get() >=
        FLAGS_tera_request_pending_limit * FLAGS_tera_quota_unlimited_pending_ratio) {
      if (!quota_entry_->CheckAndConsume(
              request->tablet_name(),
              quota::OpTypeAmountList{std::make_pair(kQuotaReadReqs, row_num)})) {
        response->set_sequence_id(request->sequence_id());
        response->set_status(kQuotaLimited);
        read_quota_rejest_counter.Add(row_num);
        VLOG(20) << "quota_entry check failed for ReadTablet";
        done->Run();
        return;
      }
    }
    read_pending_counter.Add(row_num);
    ReadRpcTimer* timer = new ReadRpcTimer(request, response, done, start_micros);
    RpcTimerList::Instance()->Push(timer);

    ReadRpc* rpc = new ReadRpc(controller, request, response, done, timer, start_micros);
    // rpc调度模块进行调度
    // 回调函数DoScheduleRpc 内部调用DoReadTablet
    read_rpc_schedule_->EnqueueRpc(request->tablet_name(), rpc);
    read_thread_pool_->AddTask(
        std::bind(&RemoteTabletNode::DoScheduleRpc, this, read_rpc_schedule_.get()));
  }
}

void RemoteTabletNode::DoReadTablet(google::protobuf::RpcController* controller, int64_t start_micros, const ReadTabletRequest* request, ReadTabletResponse* response, google::protobuf::Closure* done, ReadRpcTimer* timer) {
  int32_t row_num = request->row_info_list_size();
  read_pending_counter.Sub(row_num);

  // 调度到的时候判断是否超时，已经剩余超时时间
  bool is_read_timeout = false;
  if (request->has_client_timeout_ms()) {
    int64_t read_timeout = request->client_timeout_ms() * 1000;  // ms -> us
    int64_t detal = get_micros() - start_micros;
    if (detal > read_timeout) {
      is_read_timeout = true;
    }
  }

  if (!is_read_timeout) {
    // 核心逻辑
    tabletnode_impl_->ReadTablet(start_micros, request, response, done, read_thread_pool_.get());
  } else {
    response->set_sequence_id(request->sequence_id());
    response->set_success_num(0);
    response->set_status(kTableIsBusy);
    read_reject_counter.Inc();
    done->Run();
  }

  if (NULL != timer) {
    RpcTimerList::Instance()->Erase(timer);
    delete timer;
  }
  VLOG(8) << "finish RPC (ReadTablet)";
}
```



### TabletNodeImpl::ReadTablet

ReadTablet内部逻辑交简单，直接通过参数初始化ReadTabletTask，然后调用StartRead，下面直接看逻辑（ReadTabletTask和ShardRequest成员变量较好理解，也不再赘述）。

```c++
void ReadTabletTask::StartRead() {
  if (total_row_num_ == 0) {
    response_->set_status(kTabletNodeOk);
    response_->set_success_num(read_success_num_.Get());
    done_->Run();
    return;
  }

  response_->mutable_detail()->mutable_status()->Reserve(total_row_num_);
  for (int i = 0; i != total_row_num_; ++i) {
    response_->mutable_detail()->mutable_status()->AddAlreadyReserved();
  }

  // 默认最多并行10个任务
  // 默认一个任务最少读30个row_key
  int64_t max_task_num = FLAGS_tera_tabletnode_parallel_read_task_num;
  int64_t min_rows_per_task = FLAGS_tera_tabletnode_parallel_read_rows_per_task;
  int64_t max_size = max_task_num * min_rows_per_task;
  int64_t rows_per_task;

  /*
   * rows_per_task 既每个任务读的row_key数量
   * 小于min_rows_per_task为min_rows_per_task
   * 最大任务书为小于等于1，则为total_row_num_
   * 否则平均分
   */
  if (max_size >= total_row_num_) {
    rows_per_task = min_rows_per_task;
  } else {
    if (max_task_num <= 1) {
      rows_per_task = total_row_num_;
    } else {
      rows_per_task = total_row_num_ / max_task_num + 1;
    }
  }
  // 拆分的任务数，一个任务对应一个ShardRequest
  int64_t shard_cnt = total_row_num_ / rows_per_task + 1;

  row_results_list_.reserve(shard_cnt);

  int64_t row_to_read = total_row_num_;
  int64_t offset = 0;
  while (row_to_read > 0) {
    row_results_list_.emplace_back();
    auto shard_request = make_shared<ShardRequest>(offset, std::min(rows_per_task, row_to_read),
                                                   &row_results_list_.back());
    row_to_read -= rows_per_task;
    offset += rows_per_task;
    // We split one read request to several shard_request.
    // row_to_read <= 0 means this is the last sharded request(No more rows need
    // to read).
    // So this sharded request is processed in current thread for reducing cost
    // of switching thread.
    // Otherwise, shard_request is added to read_thread_pool.
    if (row_to_read <= 0) {
      DoRead(shard_request);
    } else {
      read_thread_pool_->AddTask(
          std::bind(&ReadTabletTask::DoRead, shared_from_this(), shard_request));
    }
  }
}

void ReadTabletTask::DoRead(std::shared_ptr<ShardRequest> shard_req) {
  bool is_timeout{false};

  auto& row_results = *shard_req->row_results;
  int64_t index = shard_req->offset;
  int64_t end_index = index + shard_req->row_num;

  // 遍历一个key 一个key去读，注意这个是user_key，如果是非kv会是多个key的聚合结果
  while (index < end_index) {
    int64_t time_remain_ms = end_time_ms_ - GetTimeStampInMs();
    StatusCode row_status = kTabletNodeOk;

    io::TabletIO* tablet_io = tablet_manager_->GetTablet(
        request_->tablet_name(), request_->row_info_list(index).key(), &row_status);
    if (tablet_io == NULL) {
      response_->mutable_detail()->mutable_status()->Set(index, kKeyNotInRange);
      read_range_error_counter.Inc();
    } else {
      row_results.emplace_back(new RowResult{});
      // 读一个user_key的核心逻辑
      if (tablet_io->ReadCells(request_->row_info_list(index), row_results.back().get(), snapshot_id_, &row_status, time_remain_ms)) {
        read_success_num_.Inc();
      } else {
        if (row_status != kKeyNotExist && row_status != kRPCTimeout) {
          if (row_status == kTabletNodeIsBusy) {
            read_reject_counter.Inc();
          } else {
            read_error_counter.Inc();
          }
        }
        row_results.pop_back();
      }
      tablet_io->DecRef();
      response_->mutable_detail()->mutable_status()->Set(index, row_status);
    }
    if (row_status == kRPCTimeout || has_timeout_.load()) {
      is_timeout = true;
      LOG(WARNING) << "seq_id: " << request_->sequence_id() << " timeout,"
                   << " clinet_timeout_ms: " << request_->client_timeout_ms();
      break;
    }
    ++index;
  }

  if (is_timeout) {
    has_timeout_.store(true);
  }

  // 结果处理
  FinishShardRequest(shard_req);
}

void ReadTabletTask::FinishShardRequest(const std::shared_ptr<ShardRequest>& shard_req) {
  // 读取结果统计是否为总读取的row 数
  if (finished_.Add(shard_req->row_num) == total_row_num_) {
    if (has_timeout_.load()) {
      response_->set_status(kRPCTimeout);
      done_->Run();
      return;
    }

    int64_t size = 0;
    for (const auto& row_results : row_results_list_) {
      size += row_results.size();
    }

    response_->mutable_detail()->mutable_row_result()->Reserve(size);
    for (auto& row_results : row_results_list_) {
      for (auto result : row_results) {
        response_->mutable_detail()->add_row_result()->Swap(result.get());
      }
    }
    response_->set_status(kTabletNodeOk);
    response_->set_success_num(read_success_num_.Get());
    done_->Run();
  }
  return;
}
```



### TabletIO::ReadCells

```c++
bool TabletIO::ReadCells(const RowReaderInfo& row_reader, RowResult* values, uint64_t snapshot_id,
                         StatusCode* status, int64_t timeout_ms) {
  {
    // 状态检查
    MutexLock lock(&mutex_);
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

  int64_t start_read_us = get_micros();

  // 如果是kv，不会有组合key的问题，只是单点的读
  if (kv_only_) {
    std::string key(row_reader.key());
    std::string value;
    if (RawKeyType() == TTLKv) {
      key.append(8, '\0');
    }
    // 内部组织ReadOptions，然后调用leveldb的get接口，较简单不再赘述
    if (!Read(key, &value, snapshot_id, status)) {
      counter_.read_rows.Inc();
      row_read_count.Inc();
      row_read_delay.Add(get_micros() - start_read_us);
      {
        MutexLock lock(&mutex_);
        db_ref_count_--;
      }
      return false;
    }
    KeyValuePair* result = values->add_key_values();
    result->set_key(row_reader.key());
    result->set_value(value);
    counter_.read_rows.Inc();
    row_read_count.Inc();
    counter_.read_size.Add(result->ByteSize());
    row_read_bytes.Add(result->ByteSize());
    row_read_delay.Add(get_micros() - start_read_us);
    {
      MutexLock lock(&mutex_);
      db_ref_count_--;
    }
    return true;
  }

  ScanOptions scan_options;
  scan_options.enable_dfs_read_thread_limiter = FLAGS_enable_dfs_read_thread_limiter;
  /*
   * ll_seek_available 标志位设置为false的逻辑较为重要
   * 1. col 为空，既user_key所在行的所有列都需要读写
   * 2. 或者有一个col下qua为空，既一个col下面全面的行都要读
   * ll_seek_available 为false调用LowLevelScan接口（见Scan介绍，底层复用了BatchScan的逻辑）
   * 本部分重点看一下LowLevelSeek 逻辑 以及与LowLevelScan的不同
   */
  bool ll_seek_available = true;
  for (int32_t i = 0; i < row_reader.cf_list_size(); ++i) {
    const ColumnFamily& column_family = row_reader.cf_list(i);
    const std::string& column_family_name = column_family.family_name();
    std::set<std::string>& qualifier_list = scan_options.column_family_list[column_family_name];
    qualifier_list.clear();
    for (int32_t j = 0; j < column_family.qualifier_list_size(); ++j) {
      qualifier_list.insert(column_family.qualifier_list(j));
    }
    if (qualifier_list.empty()) {
      ll_seek_available = false;
    }
    scan_options.iter_cf_set.insert(column_family_name);
  }
  if (scan_options.column_family_list.empty()) {
    ll_seek_available = false;
  }

  if (row_reader.has_max_version()) {
    scan_options.max_versions = row_reader.max_version();
  }

  if (row_reader.has_max_qualifiers()) {
    scan_options.max_qualifiers = row_reader.max_qualifiers();
  } else {
    scan_options.max_qualifiers = std::numeric_limits<uint64_t>::max();
  }

  if (row_reader.has_time_range()) {
    scan_options.ts_start = row_reader.time_range().ts_start();
    scan_options.ts_end = row_reader.time_range().ts_end();
  }

  scan_options.snapshot_id = snapshot_id;
  scan_options.timeout = timeout_ms;

  bool ret = false;
  // if read all columns, use LowLevelScan
  if (ll_seek_available) {
    // 见下面重点介绍
    ret = LowLevelSeek(row_reader.key(), scan_options, values, status);
  } else {
    std::string start_tera_key;
    key_operator_->EncodeTeraKey(row_reader.key(), "", "", kLatestTs, leveldb::TKT_VALUE,
                                 &start_tera_key);
    // 注意这个end_key的设置，表示是该行的数据，不会跨行
    std::string end_row_key = row_reader.key() + '\0';
    uint32_t read_row_count = 0;
    uint32_t read_cell_count = 0;
    uint32_t read_bytes = 0;
    bool complete = false;
    ret = LowLevelScan(start_tera_key, end_row_key, scan_options, values, NULL, &read_row_count,
                       &read_cell_count, &read_bytes, &complete, status);
  }
  counter_.read_rows.Inc();
  row_read_count.Inc();
  row_read_delay.Add(get_micros() - start_read_us);
  {
    MutexLock lock(&mutex_);
    db_ref_count_--;
  }
  if (!ret) {
    return false;
  } else {
    counter_.read_size.Add(values->ByteSize());
    row_read_bytes.Add(values->ByteSize());
  }

  if (values->key_values_size() == 0) {
    SetStatusCode(kKeyNotExist, status);
    return false;
  }
  return true;
}
```



##### TabletIO::LowLevelSeek

LowLevelSeek的核心逻辑，用迭代器定位到user_key的开始，然后迭代先过滤是否是所在的col，然后过滤是否是所在的qua。

和LowLevelScan相比LowLevelSeek使用了seek 没有进行批量的扫描，针对个别col和个别qua进行了优化。

###### 问题 ：如果访问多个col，只有一个col是访问全qua，其他的只有一个col，性能提升很大吗？

```c++
bool TabletIO::LowLevelSeek(const std::string& row_key, const ScanOptions& scan_options, RowResult* values, StatusCode* status) {
  StatusCode s;
  SetStatusCode(kTabletNodeOk, &s);
  values->clear_key_values();

  // create tera iterator
  leveldb::ReadOptions read_option(&ldb_options_);
  read_option.verify_checksums = FLAGS_tera_leveldb_verify_checksums;
  // 主要是根据cf 锁定需要遍历的lg
  SetupIteratorOptions(scan_options, &read_option);
  uint64_t snapshot_id = scan_options.snapshot_id;
  if (snapshot_id != 0) {
    if (!SnapshotIDToSeq(snapshot_id, &read_option.snapshot)) {
      TearDownIteratorOptions(&read_option);
      SetStatusCode(kSnapshotNotExist, status);
      return false;
    }
  }
  read_option.rollbacks = rollbacks_;
  
  // 根据key编辑码，对 start_key 和 end_key 进行编码
  // 对start_key = row_key + '\1'  , end_key = row_key + '\0'
  SetupSingleRowIteratorOptions(row_key, &read_option);
  std::unique_ptr<leveldb::Iterator> it_data(db_->NewIterator(read_option));
  TearDownIteratorOptions(&read_option);
  if (it_data->status().IsShutdownInProgress()) {
    SetStatusCode(kKeyNotInRange, status);
    return false;
  }

  // init compact strategy
  leveldb::CompactStrategy* compact_strategy = ldb_options_.compact_strategy_factory->NewInstance();

  // seek to the row start & process row delete mark
  std::string row_seek_key;
  key_operator_->EncodeTeraKey(row_key, "", "", kLatestTs, leveldb::TKT_FORSEEK, &row_seek_key);
  
  // leveldb 层的seek，提升了逐个遍历的性能
  it_data->Seek(row_seek_key);
  counter_.low_read_cell.Inc();
  low_level_read_count.Inc();
  if (it_data->Valid()) {
    leveldb::Slice cur_row_key;
    key_operator_->ExtractTeraKey(it_data->key(), &cur_row_key, NULL, NULL, NULL, NULL);
    
    // 第一个col对应的key大于row_key，表示该row_key没有数据
    if (cur_row_key.compare(row_key) > 0) {
      SetStatusCode(kKeyNotExist, &s);
    } else {
      compact_strategy->ScanDrop(it_data->key(), 0);
    }
  } else if (it_data->status().ok()) {
    SetStatusCode(kKeyNotExist, &s);
  } else {
    SetStatusCode(it_data->status(), &s);
  }

  if (s != kTabletNodeOk) {
    delete compact_strategy;
    SetStatusCode(s, status);
    return false;
  }

  // 遍历col，校验key是否在col中
  ColumnFamilyMap::const_iterator it_cf = scan_options.column_family_list.begin();
  for (; it_cf != scan_options.column_family_list.end(); ++it_cf) {
    const std::string& cf_name = it_cf->first;
    const std::set<std::string>& qu_set = it_cf->second;

    // seek to the cf start & process cf delete mark
    std::string cf_seek_key;
    key_operator_->EncodeTeraKey(row_key, cf_name, "", kLatestTs, leveldb::TKT_FORSEEK,
                                 &cf_seek_key);
    
    // seek到指定col的位置
    it_data->Seek(cf_seek_key);
    counter_.low_read_cell.Inc();
    low_level_read_count.Inc();
    if (it_data->Valid()) {
      leveldb::Slice cur_row, cur_cf;
      key_operator_->ExtractTeraKey(it_data->key(), &cur_row, &cur_cf, NULL, NULL, NULL);
      // row_key在col内没有数据
      if (cur_row.compare(row_key) > 0 || cur_cf.compare(cf_name) > 0) {
        continue;
      } else {
        compact_strategy->ScanDrop(it_data->key(), 0);
      }
    } else if (it_data->status().ok()) {
      VLOG(10) << "ll-seek fail, not found.";
      continue;
    } else {
      SetStatusCode(it_data->status(), status);
      return false;
    }

    // 这种情况是读取col下面全部qua，在上面过滤过了，该情况会调用LowLevelScan
    if (qu_set.empty()) {
      LOG(FATAL) << "low level seek only support qualifier read.";
    }
    
    // 遍历qua，检查key是否属于需要的qua
    std::set<std::string>::iterator it_qu = qu_set.begin();
    for (; it_qu != qu_set.end(); ++it_qu) {
      const std::string& qu_name = *it_qu;

      // seek to the cf start & process cf delete mark
      std::string qu_seek_key;
      key_operator_->EncodeTeraKey(row_key, cf_name, qu_name, kLatestTs, leveldb::TKT_FORSEEK, &qu_seek_key);
      
      // seek到col下相应的qua的第一个key
      it_data->Seek(qu_seek_key);
      uint32_t version_num = 0;
      for (; it_data->Valid();) {
        if (db_->IsShutdown1Finished()) {
          // break early on waiting_for_shutdown2_
          // igrone haven't scan versions of this qualifier
          TABLET_UNLOAD_LOG << "break lowlevelscan before iterator next";
          SetStatusCode(kKeyNotInRange, &s);
          break;
        }
        counter_.low_read_cell.Inc();
        low_level_read_count.Inc();
        
        leveldb::Slice cur_row, cur_cf, cur_qu;
        int64_t timestamp;
        key_operator_->ExtractTeraKey(it_data->key(), &cur_row, &cur_cf, &cur_qu, &timestamp, NULL);
        
        // 先比较col，在比较qua，大于说明遍历的数据在需要的数据后面，既没有需要的数据
        if (cur_row.compare(row_key) > 0 || cur_cf.compare(cf_name) > 0 ||
            cur_qu.compare(qu_name) > 0) {
          break;
        }

        // skip qu delete mark and out-of-range version
        if (compact_strategy->ScanDrop(it_data->key(), 0)) {
          scan_drop_count.Inc();
          // skip to next qualifier
          break;
        }

        // 校验读取的数据是否是请求需要的时间段内
        if (scan_options.ts_start > timestamp) {
          break;
        }
        if (scan_options.ts_end < timestamp) {
          it_data->Next();
          continue;
        }

        // version filter
        if (++version_num > scan_options.max_versions) {
          break;
        }

        // 设置结果
        KeyValuePair* kv = values->add_key_values();
        kv->set_key(row_key);
        kv->set_column_family(cf_name);
        kv->set_qualifier(qu_name);
        kv->set_timestamp(timestamp);

        int64_t merged_num;
        std::string merged_value;
        // ReadAble 和 binary都有多key合并的情况，BatchScan有介绍
        bool has_merged =
            compact_strategy->ScanMergedValue(it_data.get(), &merged_value, &merged_num);
        if (has_merged) {
          counter_.low_read_cell.Add(merged_num - 1);
          low_level_read_count.Add(merged_num - 1);
          kv->set_value(merged_value);
        } else {
          leveldb::Slice value = it_data->value();
          kv->set_value(value.data(), value.size());
          it_data->Next();
        }
      }
      if (!it_data->status().ok()) {
        SetStatusCode(it_data->status(), status);
        return false;
      }
      if (s == kKeyNotInRange) {
        // only on waiting_for_shutdown2_ != NULL will break with kKeyNotInRange
        // igrone haven't scan qualifiers
        break;
      }
    }
    if (s == kKeyNotInRange) {
      // only on waiting_for_shutdown2_ != NULL will break with kKeyNotInRange
      // igrone haven't scan column_families
      break;
    }
  }
  delete compact_strategy;

  SetStatusCode(s, status);
  return kTabletNodeOk == s;
}
```

