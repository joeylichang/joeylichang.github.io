# TabletNode Batch Scan

BacthScan指的是分多个请求进行遍历，一次遍历一部分而不是只遍历一部分（tera也支持部分遍历），所以TabletNodeServer需要记录BatchScan的一些（不同请求之间）中间状态（ScanContext）。

### ScanContext调度机制

1. ScanContextManager负责管理所有的ScanContext（记录BatchScan的中间状态），内部使用leveldb::Cache缓存ScanContext，缓存的key是请求中的参数session_id，session_id也是区分是否是BatchScan的标志，BacthScan的所有请求公用一个session_id。
2. 每个子请求都会调用ScanContextManager的GetScanContext从缓存中获取ScanContext，如果是第一个请求会初始化一个新的ScanContext加入到缓存中，否则将请求加入到ScanContext的job队列中然后之间返回。
3. GetScanContext之后会调用ScanContextManager的ScheduleScanContext，循环从job队列中取出请求调用TabletIo的ProcessScan（BatchScan的核心逻辑）进行遍历，然后回填response，如果遍历完成还做做一些析构的处理。



### 主要的数据结构

```c++
typedef std::pair<ScanTabletResponse*, google::protobuf::Closure*> ScanJob;
struct ScanContext {
  int64_t session_id;     // 所有请求公用的id
  TabletIO* tablet_io;    // 遍历的tabletio

  // use for lowlevelscan
  std::string start_tera_key;  // 遍历的开始key（根据key类型进行了编码）
  std::string end_row_key;     // 遍历的结束key（根据key类型进行了编码）
  ScanOptions scan_options;    // 遍历的一些配置，来自于请求（request），见下面介绍
  leveldb::Iterator* it;       // init to NULL，第一次需要new一个lelveldb的迭代器，后面介绍
  leveldb::CompactStrategy* compact_strategy;  // 压缩策略，主要用于判断key是否过期等后面介绍
  uint32_t version_num;        // 相同key（user_key col qua相同）的版本计数器与用户指定的版本数进行比较，超过用户指定的最大值，后面的略过，根据比较器看从高版本开始遍历
  uint64_t qu_num;             // qua的遍历个数，找过设置闸值max_qualifiers，结束当前循环
  std::string last_key;        // 上次遍历的key
  std::string last_col;        // 上次遍历的col
  std::string last_qual;       // 上次遍历的qua

  // use for reture
  StatusCode ret_code;  // set by lowlevelscan, 状态码
  bool complete;        // test this flag know whether scan finish or not，所有遍历是否结束
  RowResult* result;    // scan result for one round,
  uint64_t data_idx;    // return data_id， 记录batchscan的次数
  uint32_t cell_count;  // scan total cell count for one round, kvtable cell_count equal row_count （非kv 合并的数据算多条）
  uint32_t row_count;   // scan total row count for one round
  uint32_t data_size;   // scan total data size for one round

  // protect by manager lock
  std::queue<ScanJob> jobs;        // 任务队列
  leveldb::Cache::Handle* handle; 
};
```



```c++
typedef std::map<std::string, std::set<std::string> > ColumnFamilyMap;
struct ScanOptions {
  uint32_t max_versions; // 遍历的最大版本数，例如，一个被遍历到的key不同版本（时间戳），从高版本开始遍历
  uint32_t max_size;     // 请求的key val的大小最大值
  int64_t number_limit;  // kv number > number_limit, return to user
  int64_t ts_start;      // rowkey时间戳的下限
  int64_t ts_end;        // rowkey时间戳的上限
  uint64_t snapshot_id;  // 快照id
  filter::FilterPtr filter;  // 用户指定的 col qua的与或并、val大于小于等过滤条件
  ColumnFamilyMap column_family_list;  // col_name -> set<qualifiers> 的映射
  std::set<std::string> iter_cf_set;   // 所有的col_name
  filter::ColumnSet filter_column_set; //
  int64_t timeout;          // 超时时间，默认 max / 2
  uint64_t max_qualifiers;  // 遍历相同rowkey col下面 不同qua的最大数目
  // If sdk uses batch scan, we will use prefetch scan iterator.;
  bool is_batch_scan;     // 是否预读，始终未true
  bool enable_dfs_read_thread_limiter; // 是否启动 限流器，访问dfs的流控策略 见leveldb的介绍

  ScanOptions()
      : max_versions(std::numeric_limits<uint32_t>::max()),
        max_size(std::numeric_limits<uint32_t>::max()),
        number_limit(std::numeric_limits<int64_t>::max()),
        ts_start(kOldestTs),
        ts_end(kLatestTs),
        snapshot_id(0),
        timeout(std::numeric_limits<int64_t>::max() / 2),
        max_qualifiers(std::numeric_limits<uint64_t>::max()),
        is_batch_scan(false),
        enable_dfs_read_thread_limiter(false) {}
};
```



### TabletIO::HandleScan

```c++
bool TabletIO::HandleScan(const ScanTabletRequest* request, ScanTabletResponse* response, google::protobuf::Closure* done) {
  // concurrency control, ensure only one scanner step init leveldb::Iterator
  // 获取缓存中的ScanContext，没有则初始化一个并加入缓存中
  ScanContext* context = scan_context_manager_->GetScanContext(this, request, response, done);
  // 如果是null，表示有ScanContext，已经将请求加入ScanContext的job队列中待处理了
  if (context == NULL) {
    return true;
  }

  // first rpc init iterator and scan parameter
  if (context->it == NULL) {
    // 初始化scan_options，介绍见上面，基本来自于请求中
    SetupScanRowOptions(request, &(context->scan_options));
    context->scan_options.is_batch_scan = true;
    // context->complete set false in GetScanContext()
    // 根据key的类型对对key进行编码，返回编码之后的结构
    // 开始key 和 结束key 参考两部分，一个是tablet的key范围，一个是请求中的start 和 end
    SetupScanKey(request, &(context->start_tera_key), &(context->end_row_key));
    
    // 初始化Scan的迭代器，只有第一次才会初始化
    // 创建levledb的迭代器，并seek到start_key（中间涉及到key编码的转换）
    context->ret_code = InitScanIterator(context->start_tera_key, context->end_row_key,
                                         context->scan_options, &(context->it));
    if (!kv_only_ || RawKeyType() == TTLKv) {
      context->compact_strategy = ldb_options_.compact_strategy_factory->NewInstance();
    }
  }
  // schedule scan context
  // ScheduleScanContext 内部 核心逻辑调用ProcessScan
  return scan_context_manager_->ScheduleScanContext(context);
}
```



### TabletIO::ProcessScan

ProcessScan针对是否是kv类型进行了分类处理，除此之外还进行了一些统计信息。

```c++
void TabletIO::ProcessScan(ScanContext* context) {
  uint32_t rows_scan_num = 0;
  uint32_t cells_scan_num = 0;
  uint32_t size_scan_bytes = 0;
  int64_t start_scan_us = get_micros();

  if (kv_only_) {
    KvTableScan(context, &rows_scan_num, &size_scan_bytes);
    context->data_size = size_scan_bytes;
    context->cell_count = rows_scan_num;  // if kv table, cell_count equal row_count
    context->row_count = rows_scan_num;
    counter_.scan_kvs.Add(rows_scan_num);
  } else {
    LowLevelScan(context->start_tera_key, context->end_row_key, context->scan_options, context->it,
                 context, context->result, NULL, &rows_scan_num, &cells_scan_num, &size_scan_bytes,
                 &context->complete, &context->ret_code);
    context->data_size = size_scan_bytes;
    context->cell_count = cells_scan_num;
    context->row_count = rows_scan_num;
    counter_.scan_kvs.Add(cells_scan_num);
  }
  counter_.scan_rows.Add(rows_scan_num);
  counter_.scan_size.Add(size_scan_bytes);
  row_scan_count.Add(rows_scan_num);
  row_scan_bytes.Add(size_scan_bytes);
  row_scan_delay.Add(get_micros() - start_scan_us);
}
```

### 非KV遍历（Readable、Binary）

```c++
bool TabletIO::LowLevelScan(const std::string& start_tera_key, const std::string& end_row_key, const ScanOptions& scan_options, leveldb::Iterator* it, ScanContext* scan_context, RowResult* values, KeyValuePair* next_start_point, uint32_t* read_row_count, uint32_t* read_cell_count, uint32_t* read_bytes, bool* complete, StatusCode* status) {
  leveldb::CompactStrategy* compact_strategy = scan_context->compact_strategy;
  std::string& last_key = scan_context->last_key;
  std::string& last_col = scan_context->last_col;
  std::string& last_qual = scan_context->last_qual;
  uint32_t& version_num = scan_context->version_num;
  uint64_t& qu_num = scan_context->qu_num;

  SingleRowBuffer row_buf;
  uint32_t buffer_size = 0;
  int64_t number_limit = 0;
  values->clear_key_values();
  *read_row_count = 0;
  *read_cell_count = 0;
  *read_bytes = 0;
  int64_t now_time = GetTimeStampInMs();
  int64_t time_out = now_time + scan_options.timeout;
  KeyValuePair next_start_kv_pair;

  *complete = false;
  for (; it->Valid();) {
    bool has_merged = false;
    std::string merged_value;
    counter_.low_read_cell.Inc();
    low_level_read_count.Inc();
    *read_bytes += it->key().size() + it->value().size();
    ++*read_cell_count;
    now_time = GetTimeStampInMs();

    leveldb::Slice tera_key = it->key();
    leveldb::Slice value = it->value();
    leveldb::Slice key, col, qual;
    int64_t ts = 0;
    leveldb::TeraKeyType type;
    
    // tera_key 解码处 col qua user_key timesamp 类型等字段
    if (!key_operator_->ExtractTeraKey(tera_key, &key, &col, &qual, &ts, &type)) {
      it->Next();
      continue;
    }

    // 如果超时，默认超时时间无穷大，既没有超时
    // 如果next_start_point 不为空，标记下次开始的地方，batchscan未用此参数，其他流程有用到
    if (now_time > time_out) {
      if (next_start_point != NULL) {
        MakeKvPair(key, col, qual, ts, "", next_start_point);
      }
      SetStatusCode(kRPCTimeout, status);
      break;
    }
    if (db_->IsShutdown1Finished()) {
      SetStatusCode(kKeyNotInRange, status);
      break;
    }

    // 表示已完成，既已经遍历到end_row_key 后面了
    if (end_row_key.size() && key.compare(end_row_key) >= 0) {
      // scan finished
      *complete = true;
      break;
    }

    // 遍历的key的col 不再请求的col_set中
    const std::set<std::string>& cf_set = scan_options.iter_cf_set;
    if (cf_set.size() > 0 && cf_set.find(col.ToString()) == cf_set.end() &&
        type != leveldb::TKT_DEL) {
      // donot need this column, skip row deleting tag
      it->Next();
      continue;
    }

    /* drop的原则：
     * 1. 不是删除key，且col 不再schema中（schema可能有变更）
     * 2. 非添加key类型，col 过期
     * 3. 是否是删除的row
     * 4. 是否是删除的col
     * 5. 是否是删除的quas
     * 6. 是否是删除的qua
     * 7. 等等
     */
    if (compact_strategy->ScanDrop(it->key(), 0)) {
      // skip drop record
      scan_drop_count.Inc();
      it->Next();
      continue;
    }

    // only use for sync scan, not available for stream scan
    if (key_operator_->Compare(it->key(), start_tera_key) < 0) {
      // skip out-of-range records
      // keep record of version info to prevent dirty data
      if (key.compare(last_key) == 0 && col.compare(last_col) == 0 &&
          qual.compare(last_qual) == 0) {
        ++version_num;
      } else {
        last_key.assign(key.data(), key.size());
        last_col.assign(col.data(), col.size());
        last_qual.assign(qual.data(), qual.size());
        version_num = 1;
      }
      it->Next();
      continue;
    }

    // begin to scan next row
    // 表示一行遍历完毕，对一行的数据进行处理
    if (key.compare(last_key) != 0) {
      *read_row_count += 1;
      /*
       * 1. 判断是否需要过滤该行，根据用户请求中设置的参数，val是都大于或者小于一个值，col、qua之间与或并关系
       * 2. col、qua是否在请求的集合中
       * 3. 时间戳是否过期
       * 4. 最后将结果序列化到values中
       */
      ProcessRowBuffer(row_buf, scan_options, values, &buffer_size, &number_limit);
      row_buf.Clear();
    }

    // 遇到相同raw_key（user_key col qua）比较版本数是否超过请求设定的闸值
    if (key.compare(last_key) == 0 && col.compare(last_col) == 0 && qual.compare(last_qual) == 0) {
      if (++version_num > scan_options.max_versions) {
        it->Next();
        continue;
      }
    } else {
      // user_key col 相同，qua不同，检查是否超过设定的遍历qua的最大值
      if (key.compare(last_key) == 0 && col.compare(last_col) == 0) {
        if (++qu_num > scan_options.max_qualifiers) {
          it->Next();
          continue;
        }
      } else {
        qu_num = 1;
      }

      // 记录当前遍历的结果，便于下次遍历进行上面的比较逻辑
      last_key.assign(key.data(), key.size());
      last_col.assign(col.data(), col.size());
      last_qual.assign(qual.data(), qual.size());
      version_num = 1;
      int64_t merged_num = 0;
      
      /*
       * ScanMergedValue 对数据进行合并
       * 1. 合并的类型：TKT_VALUE、TKT_ADD、TKT_ADDINT64、TKT_PUT_IFABSENT、TKT_APPEND
       * 2. 内部对迭代器（it）进行++，外部无需处理
       */
      has_merged = compact_strategy->ScanMergedValue(it, &merged_value, &merged_num);
      if (has_merged) {
        counter_.low_read_cell.Add(merged_num - 1);
        low_level_read_count.Add(merged_num - 1);
        // 如果有合并的情况，value使用最终的合并值
        value = merged_value;
        key = last_key;
        col = last_col;
        qual = last_qual;
      }
    }

    // 将结果加入到row_buf中，待一行遍历结束之后调用ProcessRowBuffer（见上面）
    // row_buf 是SingleRowBuffer类型，内部维护一个RowData vector
    // RowData 记录一个user_key col qua val timestamp tpye信息
    // SingleRowBuffer 对应一行数据
    row_buf.Add(key, col, qual, value, ts);

    // ScanMergedValue may have set it->Next()
    // Must make sure has_merged == false before it->Next()
    // Couldn't put this part in if (has_merged) else { it->Next() }
    // 如果是合并值，则内部进行了迭代器++
    if (!has_merged) {
      it->Next();
    }

    // check scan buffer
    // ProcessRowBuffer 合并了一行的数据，合并的数目为number_limit
    if (buffer_size >= scan_options.max_size || number_limit >= scan_options.number_limit) {
      break;
    }
  }
  
  // context中的参数，如果完成在scan manager内部结束遍历
  *complete = !it->Valid() ? true : *complete;

  /*
   * ScanWithFilter ：检查用户请求是否制定了filter
   *
   * IsCompleteRow 
   * 1. 检测`row_buf'中的数据是否为一整行，`row_buf'为空是整行的特例，也返回true
   * 2. 从LowLevelScan的for()循环中跳出时，leveldb::Iterator* it 指向第一个不在row_buf中的cell, 如果这个cell的rowkey和row_buf中的数据rowkey相同，则说明`row_buf'中的数据不是一整行，返回false
   * 3. `row_buf'自身的逻辑保证了其中的所有cell必定属于同一行(row)
   *
   * ShouldFilterRow：针对行级别进行过滤，用户指定的规则
   * 循环中的ProcessRowBuffer 中调用ShouldFilterRowBuffer，针对col qua 以及val进行过滤，也是用户指定的
   * 只有在一整行都读取完毕时才进行行的过滤
   *
   * 不是一个完整的行且需要过滤，则跳转到下一行
   */
  if (ScanWithFilter(scan_options) && it->Valid() && !IsCompleteRow(row_buf, it) &&
      ShouldFilterRow(scan_options, row_buf, it)) {
    // 表示当前行的数据应该舍弃，跳过当前行
    GotoNextRow(row_buf, it, &next_start_kv_pair);
  } else {
    // process the last row of tablet
    // 循环中break的逻辑跳出，未调用ProcessRowBuffer可能还
    // 在这里做一个兜底，处理的是本次循环的最后一样，跳出循环之后row_buf一定是一个完整的行
    ProcessRowBuffer(row_buf, scan_options, values, &buffer_size, &number_limit);
  }

  if (*status == kRPCTimeout || *status == kKeyNotInRange) {
    return false;
  }

  if (!it->Valid() && !(it->status().ok())) {
    SetStatusCode(it->status(), status);
    return false;
  }

  SetStatusCode(kTabletNodeOk, status);
  return true;
}

```



### KV遍历

kv遍历的逻辑想多简单一些，没有涉及过滤和多key合并的问题，kv模式用户指定外，最主要的是meta_table的使用。

```c++
bool TabletIO::KvTableScan(ScanContext* scan_context, uint32_t* read_row_count,
                           uint32_t* read_bytes) {
  std::string& start = scan_context->start_tera_key;
  std::string& end = scan_context->end_row_key;
  ScanOptions& scan_options = scan_context->scan_options;
  bool& complete = scan_context->complete;
  RowResult* values = scan_context->result;
  StatusCode* status = &scan_context->ret_code;
  leveldb::CompactStrategy* compact_strategy = scan_context->compact_strategy;

  values->clear_key_values();
  *read_row_count = 0;
  *read_bytes = 0;
  int64_t now_time = GetTimeStampInMs();
  int64_t time_out = now_time + scan_options.timeout;

  auto it = scan_context->it;
  for (; it->Valid(); it->Next()) {
    leveldb::Slice key = it->key();
    leveldb::Slice value = it->value();
    ++*read_row_count;

    // 遍历到end_row_key 后面了，可以结束遍历
    if (RawKeyType() == TTLKv) {
      complete = (!end.empty() && key_operator_->Compare(key, end) >= 0);
    } else {
      complete = (!end.empty() && key.compare(end) >= 0);
    }

    now_time = GetTimeStampInMs();
    // 4 conditions: complete, timeout, max size, max row number
    if (complete || now_time >= time_out ||
        (scan_options.max_size > 0 && *read_bytes >= scan_options.max_size) ||
        *read_row_count > scan_options.number_limit) {
      break;
    }

    // 是否过期
    if (compact_strategy && compact_strategy->ScanDrop(key, 0)) {
      VLOG(10) << "[KV-Scan] key:[" << key.ToString() << "] Dropped.";
    } else {
      // 将数据写入values 既结果中
      KeyValuePair* pair = values->add_key_values();
      if (RawKeyType() == TTLKv) {
        pair->set_key(key.data(), key.size() - sizeof(int64_t));
      } else {
        pair->set_key(key.data(), key.size());
      }
      pair->set_value(value.data(), value.size());
      *read_bytes += pair->key().size() + pair->value().size();
    }

    if (db_->IsShutdown1Finished()) {
      // return early on waiting_for_shutdown2_, igrone rows haven't scan
      TABLET_UNLOAD_LOG << "break scan kv table before iterator next";
      *status = kKeyNotInRange;
      return false;
    }
  }

  if (!it->Valid()) {
    complete = true;
  }
  if (!it->Valid() && !(it->status().ok())) {
    SetStatusCode(it->status(), status);
    return false;
  }
  SetStatusCode(kTabletNodeOk, status);
  return true;
}
```

