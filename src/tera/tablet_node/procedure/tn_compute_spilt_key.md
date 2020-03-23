# TabletNodeServer Compute Split Key

Master发起spilt流程之前会先想TN发送请求计算spilt key。TN的ComputeSplitKey在调用levldb之前需要三层的逻辑，RemoteTabletNode将任务交给lightweight_ctrl_thread_pool_（10个线程）去执行，TabletNodeImpl层调用TabletIo的Split接口，前两层逻辑没什么好说的，下面看一下TabletIo的逻辑。

### TabletIO::Split

```c++
bool TabletIO::Split(std::string* split_key, StatusCode* status) {
  {
    MutexLock lock(&mutex_);
    if (status_ != kReady) {
      SetStatusCode(status_, status);
      return false;
    }
    
    // 正在压缩的tablet 不能spilt
    if (compact_status_ == kTableOnCompact) {
      SetStatusCode(kTableNotSupport, status);
      return false;
    }
    db_ref_count_++;
  }

  if (split_key->empty()) {
    std::string raw_split_key;
    /* FindSplitKey 逻辑：
     * 1. 在DBtable中对DBIpml根据大小进行排序，选择最大的DBIpml
     * 2. DBIpml 调用 Version 计算（根据传入的大小占比，此处是0.5，既大容量50%的那个文件的最大key）
     * 3. 从level 1（注意level 0 不算）开始，第一个文件的最大key开始往下逐层找到所有小于这个key的文件
     * 4. 计算这些文件大小的和，如果小于目标容量（总容量 * 0.5），则对level 1的第二个文件的最大key最为比对，逐层往下找到所有小于他的文件进行累加。
     * 5. 3-4 逐层逐个文件文件计算，直到容量超过目标容量（总容量 * 0.5），返回split_key。
     */
    if (db_->FindSplitKey(0.5, &raw_split_key)) {
      ParseRowKey(raw_split_key, split_key);
    }

    // 针对异常情况不是用容量划分
    // 直接从key范围，根据字典序找到中间key
    if (split_key->empty() || *split_key == end_key_) {
      // could not find split_key, try calc average key
      std::string smallest_key, largest_key;
      CHECK(db_->FindKeyRange(&smallest_key, &largest_key));

      std::string srow_key, lrow_key;
      if (!smallest_key.empty()) {
        ParseRowKey(smallest_key, &srow_key);
      } else {
        srow_key = start_key_;
      }
      if (!largest_key.empty()) {
        ParseRowKey(largest_key, &lrow_key);
      } else {
        lrow_key = end_key_;
      }
      FindAverageKey(srow_key, lrow_key, split_key);
    }
  }
  {
    MutexLock lock(&mutex_);
    db_ref_count_--;
  }

  if (*split_key != "" && *split_key > start_key_ && (end_key_ == "" || *split_key < end_key_)) {
    return true;
  } else {
    SetStatusCode(kTableNotSupport, status);
    return false;
  }
}
```

