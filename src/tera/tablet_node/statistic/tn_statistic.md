### RemoteTabletNode 层统计数据

```c++
read_request_counter    // read  的qps（只统计接收，无论成功与否） 注意：一个请求多个条目算多个请求
write_request_counter   // write 的qps（只统计接收，无论成功与否） 注意：一个请求多个条目算多个请求
scan_request_counter    // scan  的qps（只统计接收，无论成功与否） 注意：一次遍历算一个请求
  
read_pending_counter    // RPC 回调层等待处理的 read    注意：一个请求多个条目算多个请求
write_pending_counter   // RPC 回调层等待处理的 write   注意：一个请求多个条目算多个请求
scan_pending_counter    // RPC 回调层等待处理的 scan    注意：一次遍历算一个请求
compact_pending_counter // RPC 回调层等待处理的 compact 注意：一次压缩算一个请求
  
//由于排队请求超过闸值（10w） 或者 调度请求时已经超时 而失败的读请求QPS 注意：一个请求多个条目算多个请求
read_reject_counter 
// 由于排队请求超过闸值（10w） 或者 超过限流闸值 而失败的写请求QPS 注意：一个请求多个条目算多个请求
write_reject_counter
// 由于排队请求超过闸值（1k）而失败的sacn请求QPS，注意：一次遍历算一个请求
scan_reject_counter
  
read_quota_rejest_counter  // 由于超过quta闸值，而拒绝读请求的QPS，注意：一个请求多个条目算多个请求
write_quota_reject_counter // 由于超过quta闸值，而拒绝写请求的QPS，注意：一个请求多个条目算多个请求
scan_quota_reject_counter  // 由于超过quta闸值，而拒绝scan请求的QPS，注意：一个请求多个条目算一个请求
  
finished_read_request_counter  // RPC 返回的读请求QPS，注意：一个请求多个条目算多个请求
finished_write_request_counter // RPC 返回的写请求QPS，注意：一个请求多个条目算多个请求
finished_scan_request_counter  // RPC 返回的scan请求QPS，注意：一次遍历算一个请求
  
read_delay  // 未使用
write_delay // 未使用
scan_delay  // 未使用
  
rand_read_delay_per_request // 
write_delay_per_request     // 
scan_delay_per_request      // 
  
write_95 // 写请求的95分位统计，注意：一个请求多个条目算一个耗时，既耗时最长的
write_99 // 写请求的99分位统计，注意：一个请求多个条目算一个耗时，既耗时最长的
read_95  // 读请求的99分位统计，注意：一个请求多个条目算一个耗时，既耗时最长的
read_99  // 读请求的99分位统计，注意：一个请求多个条目算一个耗时，既耗时最长的
scan_95  // sacn请求的99分位统计
scan_99  // scan请求的99分位统计
```



### TabletNodeImpl 层统计数据

```c++
read_error_counter   // 非kKeyNotExist、kRPCTimeout、kTabletNodeIsBusy 引起的读失败的 QPS
write_error_counter  // 非kKeyNotInRange 引起的写失败的 QPS
scan_error_counter   // Scan失败的 QPS
  
read_range_error_counter  // 没有响应的TabletIO（既KeyRange错误），引起的读失败的 QPS
write_range_error_counter // 没有响应的TabletIO（既KeyRange错误），引起的写失败的 QPS
scan_range_error_counter  // 没有响应的TabletIO（既KeyRange错误），引起的Scan失败的 QPS
  
read_reject_counter  // 与RemoteTabletNode 中是同一个变量，在Impl层加上因Busy而拒绝的请求 QPS
  
TabletNodeImpl::CacheMetrics {
  block_cache_hitrate_ // leveldb block cache 缓存用户数据，用的ShardedLRUCache 详见leveldb介绍) 命中率，没秒计算一次
  block_cache_entries_ // leveldb block cache 缓存数据的个数，每秒统计一次
  block_cache_charge_  // leveldb block cache 缓存的使用量（使用的个数），每秒刷新一次
  table_cache_hitrate_ // leveldb table cache 缓存索引（用的ShardedLRUCache 详见leveldb介绍) 命中率，没秒计算一次
  table_cache_entries_ // leveldb table cache 缓存数据的个数，每秒统计一次
  table_cache_charge_  // leveldb table cache 缓存的使用量（使用的个数），每秒刷新一次
}

snappy_ratio_metric_  // 压缩的比例
```



### TabletIO 层统计数据

```c++
struct StatCounter {
    tera::MetricCounter low_read_cell; // 非KV类型遍历leveldb的一个kv对为一个cell，其QPS
    tera::MetricCounter scan_rows; // 遍历的行数（非KV类型遍历可能是多个kv对的和），其QPS
    tera::MetricCounter scan_kvs;  // 遍历的kv对（非KV类型一行对应多个kv对，KV类型一对一）其QPS
  																 // 对于KV类型，scan_rows == scan_kvs
    tera::MetricCounter scan_size; // scan 读取的数据带宽，1s更新一次（时间段的平均值）
    tera::MetricCounter read_rows; // 读的行数，kv就是读的kv对数，非KV可能一行对应的对个kv对，读请求一次只能读一行，都是+1，其QPS（注意：失败也算哦！）
    tera::MetricCounter read_kvs;  // 未使用
    tera::MetricCounter read_size; // 读取成功的带宽，1s更新一次（时间段的平均值）
    tera::MetricCounter write_rows;// 写的行号（一行可能对应多个kv对），QPS
    tera::MetricCounter write_kvs; // 写的KV对，QPS
    tera::MetricCounter write_size;// 写成功的带宽，1s更新一次（时间段的平均值）
    tera::MetricCounter write_reject_rows; // 写TabletWriter 失败的行数，1s更新一次
}

low_level_read_count // 同StatCounter 的 low_read_cell
scan_drop_count      // 由于过期、删除标记等原因，被丢弃的cell ，QPS
scan_filter_count    // 对于已经读取完整一行的数据，进行行、col、qu、时间区间因素被丢弃的行数，QPS
batch_scan_count     // BatchScan 的QPS
sync_scan_count      // 非KV，一次性遍历 的QPS
  
row_read_delay // 
row_read_count // 同StatCounter 的 read_rows
row_read_bytes // 同StatCounter 的 read_size
  
row_scan_delay // 未使用
row_scan_count // 同StatCounter 的 scan_rows
row_scan_bytes // 同StatCounter 的 scan_size
  
row_write_bytes // 同StatCounter 的 write_size
  
row_read_delay_per_row // 
row_scan_delay_per_row // 
```



### TabletWriter 层统计数据

```c++
flush_to_disk_check_delay  // 刷入磁盘之前的check耗时，非法输入、TXN冲突等，最近一次耗时情况
flush_to_disk_write_delay  // 写入leveldb的耗时，最近一次耗时情况
flush_to_disk_batch_delay  // 打包批量写请求的耗时，最近一次耗时情况
flush_to_disk_finish_delay // 结束请求调用请求回调之后的耗时，最近一次耗时情况
  
row_write_count            // 同TabletIO的StatCounter的write_rows
row_write_delay            // 未使用
row_write_delay_per_row    // 
```



### PersistentCache 层统计数据

```c++
struct Statistics {
  write_throughput // 写入的带宽，1s更新一次
  write_count      // 写入数据的 QPS
  read_throughput  // 读取的带宽，1s更新一次
  cache_hits       // PersistentCache 命中 QPS
  cache_misses     // PersistentCache 未命中的 QPS
  cache_errors     // 数据在PersistentCache，但是读取是遇到错误，QPS
  file_entries     // 
};
```



### TabletNodeSysInfo

周期性（1s）调用RefreshAndDumpSysInfo

```c++
void TabletNodeImpl::RefreshAndDumpSysInfo() {
  int64_t cur_ts = get_micros();

  // 收集上述信息（比上面要全一些，但是根据字面意思可以理解）并dump
  sysinfo_.CollectTabletNodeInfo(tablet_manager_.get(), local_addr_);
  sysinfo_.CollectHardwareInfo();
  sysinfo_.SetTimeStamp(cur_ts);
  sysinfo_.UpdateWriteFlowController();
  sysinfo_.DumpLog();

  VLOG(15) << "collect sysinfo finished, time used: " << get_micros() - cur_ts << " us.";
}

// dump信息如下
TabletNodeSysInfo::TabletNodeSysInfo()
    : info_{new TabletNodeInfo}, tablet_list_{new TabletMetaList} {
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpSysInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpHardWareInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpIoInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpCacheInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpRequestInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpDfsInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpPosixInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpLevelSizeInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpPersistentCacheInfo);
  RegisterDumpInfoFunction(&TabletNodeSysInfo::DumpOtherInfo);
}
```



Master 心跳探测收集两部分信息：GetTabletNodeInfo、GetTabletMetaList，既节点和tablet，相关统计信息全部来自埋点。

###### 注意：埋点的统计数据涉及MetricCounter、CollectorReportPublisher、Collector、Subscriber等类，逻辑相对清晰好理解，不在此赘述。
