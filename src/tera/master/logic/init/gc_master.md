# Master GC

### GC主要流程

1. master收集tablet无用的file（内存中维护的tablet信息与存储目录的tablet信息对比）——tablet级别操作。
2. 发送心跳探测，心跳探测中设置is_gc_query字段——tabletnode级别操作。
3. 心跳返回包中，根据汇报的tablet的信息与master内存中的文件信息进行对比，进行释放和添加——tablet级别的操作。
4. 待所有的返回包都执行完毕后。
5. 遍历tablet，对于未使用和删除的tablet，都移动到trash目录（trash目录有两个，后面会介绍）。
6. 周期性（默认一小时）去删除trash目录的数据，删除时只删除24小时以上的旧数据（trash目录的数据保留24小时）。

1-5流程周期性执行（默认60s），但是心跳探测可能还未返回在60s内，所以内部有同步机制，保证1-5按串行的，只有全部结束之后才会新的一轮gc。

注意：流程中设计到的存储目录，可以在启动时设定，支持本地存储、dfs等。

### GC 源码

master收集tablet无用的file在TryCollectInheritedFile。

```c++
bool Table::TryCollectInheritedFile() {
  if (GetTableName() == FLAGS_tera_master_meta_table_name) {
    return false;
  }

  std::set<uint64_t> live_tablets, dead_tablets;
  // 遍历存储目录，与内存数据对比，内存中没有但是存储目录有则家督dead_tablets
  GetTabletsForGc(&live_tablets, &dead_tablets, true);

  std::set<uint64_t>::iterator it = dead_tablets.begin();
  for (; it != dead_tablets.end(); ++it) {
    std::vector<TabletFile> tablet_files;
    /*
     * 获取tablet在存储目录的文件
     * 格式：../data/${table_name}/tablet_0001/lg/file/001.sst
     */
    CollectInheritedFileFromFilesystem(name_, *it, &tablet_files);

    if (tablet_files.empty()) {
      MutexLock l(&mutex_);
      /*
       * 加入成员gc_disabled_dead_tablets_.insert(tablet_id)
       * 清空成员变量gc_disabled_dead_tablets_[tablet_id]
       */
      AddEmptyDeadTablet(*it);
    } else {
      for (uint32_t i = 0; i < tablet_files.size(); i++) {
        MutexLock l(&mutex_);
        // 引用计数加1，后续删除时会减1，防止这期间被释放
        AddInheritedFile(tablet_files[i], false);
      }
    }
  }
  return dead_tablets.size() > 0;
}
```



心跳探测包回来之后，进行tablet内file的对比逻辑如下：

```C++
void Table::GarbageCollect(const TabletInheritedFileInfo& tablet_inh_info) {
  // sort reported files
  std::multiset<TabletFile> report_inh_files;
  for (int32_t i = 0; i < tablet_inh_info.lg_inh_files_size(); i++) {
    const LgInheritedLiveFiles& lg_inh_files = tablet_inh_info.lg_inh_files(i);
    struct TabletFile inh_file;
    inh_file.lg_id = lg_inh_files.lg_no();
    for (int32_t j = 0; j < lg_inh_files.file_number_size(); j++) {
      leveldb::ParseFullFileNumber(lg_inh_files.file_number(j), &inh_file.tablet_id,
                                   &inh_file.file_id);
      report_inh_files.insert(inh_file);
    }
  }

  MutexLock l(&mutex_);
  Table::TabletList::iterator tablet_it = tablets_list_.find(tablet_inh_info.key_start());
  if (tablet_it == tablets_list_.end()) {
    return;
  }
  TabletPtr tablet = tablet_it->second;
  if (tablet->GetKeyEnd() != tablet_inh_info.key_end()) {
    return;
  }

  // insert a MAX element to simplify two sets' comparason
  struct TabletFile max = {UINT64_MAX, INT32_MAX, UINT64_MAX};
  report_inh_files.insert(max);
  tablet->inh_files_.insert(max);
  std::multiset<TabletFile>::iterator old_it = tablet->inh_files_.begin();
  std::multiset<TabletFile>::iterator new_it = report_inh_files.begin();
  /*
   * 心跳上报的file 与 本地file进行对比（tablet内的file）
   */
  while (old_it != tablet->inh_files_.end() && new_it != report_inh_files.end()) {
    if (*old_it == *new_it) {
      ++old_it;
      ++new_it;
    } else if (*old_it < *new_it) {
      VLOG(10) << "[gc] " << tablet->GetPath() << " release file " << *old_it;
      ReleaseInheritedFile(*old_it);
      old_it = tablet->inh_files_.erase(old_it);  // desc ref for tablet->inh_files_
    } else if (!tablet->gc_reported_) {
      VLOG(10) << "[gc] " << tablet->GetPath() << " report file " << *new_it;
      // 上述有介绍
      AddInheritedFile(*new_it, true);  // inc ref for tablet->inh_files_
      tablet->inh_files_.insert(*new_it);
      ++new_it;
    } else {
      LOG(WARNING) << "[gc] ignore(query error) " << tablet->GetPath() << " report new file "
                   << *new_it;
      ++new_it;
    }
  }
  tablet->inh_files_.erase(max);

  if (!tablet->gc_reported_) {
    tablet->gc_reported_ = true;
    // 表示当前table的所有tablet已经汇总完毕，可以进行table级别的gc
    if (++reported_live_tablets_num_ == tablets_list_.size()) {
      // now all live tablets report finish
      std::set<uint64_t>::iterator it = gc_disabled_dead_tablets_.begin();
      for (; it != gc_disabled_dead_tablets_.end(); ++it) {
        /*
         * 将文件放入obsolete_inh_files_ ———— 最终要铲除的文件
         * 从useful_inh_files_中移除 ———— 相当于心跳回收阶段的临时存储
         * 如果目录为空，也将目录放进去
         */
        EnableDeadTabletGarbageCollect(*it);
      }
      gc_disabled_dead_tablets_.clear();
    }
  }
}
```

等所有的新校都返回后，遍历tablet，将文件统一放入trash目录，源码如下：

```c++
uint64_t Table::CleanObsoleteFile() {
  leveldb::Env* env = io::LeveldbBaseEnv();
  std::string table_path = FLAGS_tera_tabletnode_path_prefix + "/" + name_;
  uint64_t delete_file_num = 0;
  int64_t start_ts = get_micros();

  MutexLock l(&mutex_);
  while (!obsolete_inh_files_.empty()) {
    TabletFile file = obsolete_inh_files_.front();
    mutex_.Unlock();

    if (GetStatus() == kTableDeleting) {
      LOG(INFO) << "[gc] [" << name_ << "] table deleted, give up clean";
      mutex_.Lock();
      break;
    }

    std::string path;
    leveldb::Status s;
    if (file.lg_id == 0 && file.file_id == 0) {
      std::string path = leveldb::BuildTabletPath(table_path, file.tablet_id);
      leveldb::FileLock* file_lock = nullptr;
      // NEVER remove the trailing character '/', otherwise you will lock the
      // parent directory
      s = env->LockFile(path + "/", &file_lock);
      if (!s.ok()) {
        LOG(WARNING) << "lock path failed, path: " << path << ", status: " << s.ToString();
      }
      delete file_lock;

      LOG(INFO) << "[gc] [" << name_ << "] delete dir " << path;
      
      s = io::DeleteEnvDir(path);  // safely delete dir and all file in it
    } else {
      std::string lg_path = leveldb::BuildTabletLgPath(table_path, file.tablet_id, file.lg_id);
      leveldb::FileLock* file_lock = nullptr;
      // NEVER remove the trailing character '/', otherwise you will lock the
      // parent directory
      s = env->LockFile(lg_path + "/", &file_lock);
      if (!s.ok()) {
        LOG(WARNING) << "lock path failed, path: " << lg_path << ", status: " << s.ToString();
      }
      delete file_lock;

      std::string path =
          leveldb::BuildTableFilePath(table_path, file.tablet_id, file.lg_id, file.file_id);
      if (FLAGS_tera_master_gc_trash_enabled) {
        LOG(INFO) << "[gc] [" << name_ << "] move file to trash, file: " << file
                  << ", path: " << path;
        // move sst to trackable gc trash instead of deleting it directly
        // 将文件移至 ../data/#trackable_gc_trash 目录先
        s = io::MoveSstToTrackableGcTrash(name_, file.tablet_id, file.lg_id, file.file_id);
      } else {
        LOG(INFO) << "[gc] [" << name_ << "] delete file " << file << " path " << path;
        s = env->DeleteFile(path);
      }
    }
    mutex_.Lock();
    if (s.ok()) {
      delete_file_num++;
    } else {
      LOG(WARNING) << "[gc] fail to delete: " << path << " status: " << s.ToString();
    }
    obsolete_inh_files_.pop();
  }
  LOG(INFO) << "[gc] [" << name_ << "] clean obsolete file/dir, total: " << delete_file_num
            << ", cost: " << (get_micros() - start_ts) / 1000 << " ms";
  return delete_file_num;
}
```

在系统内部除了删除的"../data/#trackable_gc_trash"目录，还有一个trash目录"../data/#trash"在所有心跳探测回来之后会清理一次，这个目录在删除table、tablet时会用到后续介绍。