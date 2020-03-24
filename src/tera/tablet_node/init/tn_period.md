# TabletNode 周期任务

TabletNode 周期性任务分为一下几种：

1. GC：主要是PersistentCache的GC，默认配置是30min执行一次。
2. 统计信息刷新，默认1s更新一次。
3. SysInfo信息刷洗，默认1s更新一次。
4. RPC回调层，线程池、任务等信息profile，默认1s更新一次。
5. RPC最大延迟信息输出，默认1s更新一次。

### TabletNodeEntry::Run()

TabletNode 在StartServer内部初始化，之后会重复执行Run，在Run内部执行周期性任务。

```c++
bool TabletNodeEntry::Run() {
  static int64_t timer_ticks = 0;
  ++timer_ticks;

  // Run garbage collect, in secondes.
  // FLAGS_tera_garbage_collect_period 默认配置 是 1800s
  const int garbage_collect_period =
      (FLAGS_tera_garbage_collect_period) ? FLAGS_tera_garbage_collect_period : 60;
  if (timer_ticks % garbage_collect_period == 0) {
    // 见后面介绍
    tabletnode_impl_->GarbageCollect();
  }

  // 遍历所有的tablet，以及每个tablet的lg，对多有lg每层的db大小进行加和，默认关闭，调试使用
  if (FLAGS_tera_tabletnode_dump_level_size_info_enabled) {
    tabletnode_impl_->RefreshLevelSize();
  }

  // 见统计部分介绍
  CollectorReportPublisher::GetInstance().Refresh();
  tabletnode_impl_->RefreshAndDumpSysInfo();

  // RPC回调层，线程池、任务等信息profile
  LOG(INFO) << "[ThreadPool schd/task/cnt] " << remote_tabletnode_->ProfilingLog();

  int64_t now_time = get_micros();
  int64_t earliest_rpc_time = now_time;
  
  // 获取队列头，既等待时间最久的RPC请求时间
  RpcTimerList::Instance()->TopTime(&earliest_rpc_time);
  double max_delay = (now_time - earliest_rpc_time) / 1000.0;
  LOG(INFO) << "pending rpc max delay: " << std::fixed << std::setprecision(2) << max_delay;
  if (FLAGS_tera_tabletnode_hang_detect_enabled &&
      max_delay > FLAGS_tera_tabletnode_hang_detect_threshold) {
    LOG(FATAL) << "hang detected: " << std::fixed << std::setprecision(2) << max_delay;
  }

  ThisThread::Sleep(1000);
  return true;
}
```



### Tablet GC 介绍

```c++
/*
 * all cached tablets/files:
 * ------------------------------------------
 * | active tablets  |   inactive tablets   |
 * |                 |                      |
 * |                 |    all    |    to    |
 * |                 | inherited | *DELETE* |
 * |                 |    files  |          |
 * ------------------------------------------
 */
void TabletNodeImpl::GarbageCollect() {
  int64_t start_ms = get_micros();
  LOG(INFO) << "[gc] start...";

  // get all inherited sst files
  std::vector<InheritedLiveFiles> table_files;
  GetInheritedLiveFiles(table_files);
  std::set<std::string> inherited_files;
  for (size_t t = 0; t < table_files.size(); ++t) {
    const InheritedLiveFiles& live = table_files[t];
    int lg_num = live.lg_live_files_size();
    for (int lg = 0; lg < lg_num; ++lg) {
      const LgInheritedLiveFiles& lg_live_files = live.lg_live_files(lg);
      for (int f = 0; f < lg_live_files.file_number_size(); ++f) {
        std::string file_path =
            leveldb::BuildTableFilePath(live.table_name(), lg, lg_live_files.file_number(f));
        inherited_files.insert(file_path);
        // file_path : table-name/tablet-xxx/lg-num/xxx.sst
        VLOG(GC_LOG_LEVEL) << "[gc] inherited live file: " << file_path;
      }
    }
  }

  // get all active tablets
  std::vector<TabletMeta*> tablet_meta_list;
  std::set<std::string> active_tablets;
  tablet_manager_->GetAllTabletMeta(&tablet_meta_list);
  std::vector<TabletMeta*>::iterator it = tablet_meta_list.begin();
  for (; it != tablet_meta_list.end(); ++it) {
    VLOG(GC_LOG_LEVEL) << "[gc] Active Tablet: " << (*it)->path();
    active_tablets.insert((*it)->path());
    delete (*it);
  }

  // collect persistent cache garbage
  PersistentCacheGarbageCollect(inherited_files, active_tablets);

  // collect flash directories
  // 本系列关注PersistentCache，FlashEnv是旧版本在此不关注
  leveldb::FlashEnv* flash_env = (leveldb::FlashEnv*)io::LeveldbFlashEnv();
  if (flash_env) {
    const std::vector<std::string>& flash_paths = flash_env->GetFlashPaths();
    for (size_t d = 0; d < flash_paths.size(); ++d) {
      std::string flash_dir = flash_paths[d] + FLAGS_tera_tabletnode_path_prefix;
      GarbageCollectInPath(flash_dir, leveldb::Env::Default(), inherited_files, active_tablets);
    }
  }

  // 本系列文章不涉及
  leveldb::Env* mem_env = io::LeveldbMemEnv()->CacheEnv();
  GarbageCollectInPath(FLAGS_tera_tabletnode_path_prefix, mem_env, inherited_files, active_tablets);

  LOG(INFO) << "[gc] finished, time used: " << get_micros() - start_ms << " us.";
}
```



##### PersistentCacheGarbageCollect

```c++
void TabletNodeImpl::PersistentCacheGarbageCollect(const std::set<std::string>& inherited_files,
                                                   const std::set<std::string>& active_tablets) {
  std::shared_ptr<leveldb::PersistentCache> p_cache;
  if (!io::GetPersistentCache(&p_cache).ok() || !p_cache) {
    return;
  }
  leveldb::StopWatchMicro timer(leveldb::Env::Default(), true);
  std::vector<std::string> all_keys{p_cache->GetAllKeys()};
  /*
   * all cached tablets/files:
   * ------------------------------------------
   * | active tablets  |   inactive tablets   |
   * |                 |                      |
   * |                 |    all    |    to    |
   * |                 | inherited | *DELETE* |
   * |                 |    files  |          |
   * ------------------------------------------
   * We need to save active tablets' files and inherited files.
   * Try remove files of tablets not on this tabletserver.
   * Here is the gc rule:
   *
   * Key format in persistent cache:  |table_name/tablet_name/lg_num/xxxxxxxx.sst|
   *                                  |          1           |                   |
   * string in active_tablets         |table_name/tablet_name|                   |
   *                                  |                      2                   |
   * string in inherited_files        |table_name/tablet_name/lg_num/xxxxxxxx.sst|
   *
   * If part 1 of persistent cache key doesn't match any string in active tablets,
   * and part 2 of it doesn't match any one in inherited_files, we'll remove it.
   */
  // 上面的注释较为清晰
  std::unordered_set<std::string> new_delayed_gc_files;
  for (auto& key : all_keys) {
    if (inherited_files.find(key) != inherited_files.end()) {
      // 1. If file name in inherited_files, skip it.
      continue;
    }
    std::vector<std::string> splited_terms;
    SplitString(key, "/", &splited_terms);
    assert(splited_terms.size() > 2);
    // 2. Extract table_name/tablet_name from persistent key.
    std::string tablet_name = splited_terms[0] + "/" + splited_terms[1];
    if (active_tablets.find(tablet_name) != active_tablets.end()) {
      // 3. Skip active tablets' file.
      continue;
    }
    if (delayed_gc_files_.find(key) != delayed_gc_files_.end()) {
      LOG(INFO) << "[Persistent Cache GC] Remove unused file: " << key << ".";
      // 4. If this key has already be delayed for one gc period, remove it.
      p_cache->ForceEvict(key);
    } else {
      LOG(INFO) << "[Persistent Cache GC] Add file: " << key << " to delayed gc files.";
      // 5. Otherwise, it'll be add to delayed_gc_files, waiting for next gc process.
      new_delayed_gc_files.emplace(key);
    }
  }

  std::swap(delayed_gc_files_, new_delayed_gc_files);
  
  // 这是全局PersistentCache GC的入口，强制PersistentCache淘汰的数据会先加入被选择，待该函数调用时从备选集中选出来，看引用计数是否为0，再决定是否删除，详情见 levledb 中 PersistentCache 部分介绍
  p_cache->GarbageCollect();
  LOG(INFO) << "[Persistent Cache GC] Finished, cost: " << timer.ElapsedMicros() / 1000 << " ms.";
}
```

