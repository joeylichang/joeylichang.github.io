# Master 侧进行的LoadBalance

### 负载均衡策略

负载均衡策略继承自Scheduler基类，各自继承实现具体分策略，分为流量负载均衡和容量负载均衡，各自策略的回调顺序是NeedSchedule、DescendingSort、MayMoveOut、FindBestNode、FindBestTablet。下面具体看一下两种策略中每种回调的逻辑。

##### 根据qps进行负载均衡的策略

1. NeedSchedule
   1. 如果节点的pending_qps（读 + 写 + scan * 300）超过闸值（10亿），标记为pending_node
   2. 当pending_node比例超过10%（认为此事 dfs io会成为读的瓶颈），则认为需要进行负载均衡。
   3. 注意：一个节点负载过大不进行搬迁，达到一定比例之后才搬迁，认为dfs io在个别节点负载大时不算瓶颈，那搬迁之后对dfs的io 就会减少吗？集群整体压力并没有变化，怎么做到搬迁之后就减少dfs io的呢？
2. DescendingSort
   1. 比较顺序tabletnode 的 read_pending、row_read_delay、table的qps（table为空，qps为0）。
3. MayMoveOut
   1. tabletnode 的 pending_qps（读 + 写 + scan * 300）是否超过闸值（10亿）。
   2. table的qps是否大于0。
4. FindBestNode
   1. 比较顺序tabletnode 的 read_pending、row_read_delay、table的qps（table为空，qps为0）。
   2. 上述都相等时，比较addr，取较大的一个。
5. FindBestTablet
   1. tablet 的 read_rows + write_rows + scan_rows最大的tablet。

##### 根据容量进行负载均衡的策略

1. NeedSchedule
   1. 直接返回true。
   2. 注意：为什么直接允许调度呢？
2. DescendingSort
   1. table在该节点上的容量排序。
3. MayMoveOut
   1. table在该节点上的容量是否超过闸值（默认是0）。
4. FindBestNode
   1. table在该节点上的容量排序。
   2. 上述都相等时，比较addr，取较大的一个。
5. FindBestTablet
   1. 源节点最多向目标节点搬迁自身10%容量的数据。
   2. 选择tablet容量最大的，但是不能超过上述条件的tablet。

##### 根据容量均衡的策略比较宽松，一定与上层的调用逻辑密不可分。



### Master中负载均衡

了解了负载均衡策略逻辑之后，看一下在master中回调的调用环境和逻辑。

1. Master在心跳探测一轮结束后会进行负载均衡（如果前一轮负载均衡没有结束，则不进行下一轮）。
2. 先进行qps策略的负载均衡，然后进行容量的负载均衡，容量的负载均衡可以指定table（不指定时，默认为为空）。
3. 一轮负载均衡最多搬迁50个tablet。

### TabletNodeLoadBalance

```C++
bool MasterImpl::TabletNodeLoadBalance(TabletNodePtr tabletnode, Scheduler *scheduler,
                                       const std::vector<TabletPtr> &tablet_list,
                                       const std::string &table_name) {
  VLOG(7) << "TabletNodeLoadBalance() " << tabletnode->GetAddr() << " " << scheduler->Name() << " "
          << table_name;
  if (tablet_list.size() < 1) {
    return false;
  }

  bool any_tablet_split = false;
  std::vector<TabletPtr> tablet_candidates;

  // ts节点上面所有的tablet
  std::vector<TabletPtr>::const_iterator it;
  for (it = tablet_list.begin(); it != tablet_list.end(); ++it) {
    TabletPtr tablet = *it;
    if (tablet->GetStatus() != TabletMeta::kTabletReady ||
        tablet->GetTableName() == FLAGS_tera_master_meta_table_name) {
      continue;
    }
    double write_workload = tablet->GetCounter().write_workload();
    
    // 默认值512MB
    int64_t split_size = FLAGS_tera_master_split_tablet_size;
    if (tablet->GetSchema().has_split_size() && tablet->GetSchema().split_size() > 0) {
      split_size = tablet->GetSchema().split_size();
    }
    
    // 默认和9999，除了考虑容量 还要 考虑负载
    if (write_workload > FLAGS_tera_master_workload_split_threshold) {
      // 默认值64
      if (split_size > FLAGS_tera_master_min_split_size) {
        // 超过64M，拆分大小为 0.5 * 512 = 256MB（默认情况下）
        split_size = std::max(FLAGS_tera_master_min_split_size,
                              static_cast<int64_t>(split_size * FLAGS_tera_master_min_split_ratio));
      }
      VLOG(6) << tablet->GetPath() << ", trigger workload split, write_workload: " << write_workload
              << ", split it by size(M): " << split_size;
    }
    
    // 默认值是0
    int64_t merge_size = FLAGS_tera_master_merge_tablet_size;
    if (tablet->GetSchema().has_merge_size() && tablet->GetSchema().merge_size() > 0) {
      merge_size = tablet->GetSchema().merge_size();
    }
    // 根据split_size 设置merge_size，并发送给ts
    if (merge_size == 0) {
      int64_t current_time_s = static_cast<int64_t>(time(NULL));
      int64_t table_create_time_s =
          static_cast<int64_t>(tablet->GetTable()->CreateTime() / 1000000);
      if (current_time_s - table_create_time_s >= FLAGS_tera_master_disable_merge_ttl_s &&
          tablet->GetTable()->LockTransition()) {
        int64_t new_split_size = tablet->GetSchema().split_size();
        if (new_split_size > FLAGS_tera_master_max_tablet_size_M) {
          new_split_size = FLAGS_tera_master_max_tablet_size_M;
        }
        int64_t new_merge_size = new_split_size >> 2;
        UpdateTableRequest *request = new UpdateTableRequest();
        UpdateTableResponse *response = new UpdateTableResponse();
        TableSchema *schema = request->mutable_schema();
        schema->CopyFrom(tablet->GetSchema());
        schema->set_split_size(new_split_size);
        schema->set_merge_size(new_merge_size);
        google::protobuf::Closure *closure = UpdateDoneClosure::NewInstance(request, response);
        std::shared_ptr<Procedure> proc(new UpdateTableProcedure(
            tablet->GetTable(), request, response, closure, thread_pool_.get()));
        MasterEnv().GetExecutor()->AddProcedure(proc);

        merge_size = new_merge_size;
        VLOG(6) << "table: " << tablet->GetTableName()
                << " enable merge after ttl_s: " << FLAGS_tera_master_disable_merge_ttl_s
                << " current_time_s: " << current_time_s
                << " table_create_time_s: " << table_create_time_s
                << " try set split size(M) to be: " << new_split_size
                << " try set merge size(M) to be: " << merge_size;
      } else {
        VLOG(20) << "table: " << tablet->GetTableName()
                 << " remain disable merge in ttl_s: " << FLAGS_tera_master_disable_merge_ttl_s
                 << " current_time_s: " << current_time_s
                 << " table_create_time_s: " << table_create_time_s;
      }
    }
    if (tablet->GetDataSize() < 0) {
      // tablet size is error, skip it
      continue;
      
      /*
       * 拆分tablet时，只考虑容量
       * 合并tablet时，除了考虑容量还要考虑负载不能过高（原则应该是不影响业务请求）
       */
    } else if (tablet->GetDataSize() > (split_size << 20) &&
               tablet->TestAndSetSplitTimeStamp(get_micros())) {
      TrySplitTablet(tablet);
      any_tablet_split = true;
      continue;
    } else if (tablet->GetDataSize() < (merge_size << 20)) {
      if (!tablet->IsBusy() && write_workload < FLAGS_tera_master_workload_merge_threshold) {
        TryMergeTablet(tablet);
      } else {
        VLOG(6) << "[merge] skip high workload tablet: " << tablet->GetPath() << ", write_workload "
                << write_workload;
      }
      continue;
    }
    if (tablet->GetStatus() == TabletMeta::kTabletReady) {
      tablet_candidates.push_back(tablet);
    }
  }

  // if any tablet is splitting, no need to move tablet
  if (!FLAGS_tera_master_move_tablet_enabled || any_tablet_split) {
    return false;
  }

  TabletNodePtr dest_tabletnode;
  size_t tablet_index = 0;
  if (scheduler->MayMoveOut(tabletnode, table_name) &&
      tabletnode_manager_->ScheduleTabletNode(scheduler, table_name, nullptr, true,
                                              &dest_tabletnode) &&
      tabletnode_manager_->ShouldMoveData(scheduler, table_name, tabletnode, dest_tabletnode,
                                          tablet_candidates, &tablet_index) &&
      dest_tabletnode->GetState() == kReady) {
    TryMoveTablet(tablet_candidates[tablet_index], dest_tabletnode);
    return true;
  }
  return false;
}
```

1. 负载均衡除了tablet的搬迁，还有一个任务是tablet的spilt 和 合并（逻辑见代码注释）。
2. ScheduleTabletNode：选取目标节点。
   1. 节点必须是kReady状态。
   2. tablet的FlashSize够用，则不能搬迁。
   3. 节点刚刚完成tablet加载，则不能搬迁。
   4. 节点正在搬迁，则不能在进行搬迁。
      1. 记录节点最近20次的搬迁时间。
      2. 第一次搬迁时间的时间间隔超过300s，则认为可以进行下一次的搬迁。
   5. 节点的平均pending_read > 100，则不能搬迁。
   6. 如果候选集为空，则可以选meta_table所在的节点进行搬迁。
3. ShouldMoveData选取目标tablet。
4. TryMoveTablet：搬迁slot后续介绍。
