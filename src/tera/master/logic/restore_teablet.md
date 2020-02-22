# Master 加载MeatTablet数据

Master加载完ts之后，会进行meat_tablet的加载。tablet的加载第一步找到meta_tablet（既在zk中tera/root_table的节点存储），第二部是根据meta_tablet加载table（级别）的原信息。

### RestoreMetaTablet

```C++
bool MasterImpl::RestoreMetaTablet(const std::vector<TabletMeta> &tablet_list) {
  bool loaded_twice = false;
  bool loaded = false;
  TabletMeta meta_tablet_meta;
  std::vector<TabletMeta>::const_iterator it = tablet_list.begin();
  
  /*
   * 遍历所有的tablet，找到表明为meta_table的tablt
   * 1. meta_tablet is loaded by more than one tabletnode, unload them all
   * 2. meta_tablet is incomplete (not from "" to ""), unload it
   */
  for (; it != tablet_list.end(); ++it) {
    StatusCode status = kTabletNodeOk;
    const TabletMeta &meta = *it;
    if (meta.table_name() == FLAGS_tera_master_meta_table_name) {
      const std::string &key_start = meta.key_range().key_start();
      const std::string &key_end = meta.key_range().key_end();
      if (loaded_twice) {
        // 向ts发送请求，卸载tablet
        if (!UnloadTabletSync(FLAGS_tera_master_meta_table_name, key_start, key_end,
                              meta.server_addr(), &status)) {
          // 将ts节点加入到zk 的 /tera/kick 的子节点中
          TryKickTabletNode(meta.server_addr());
        }
      } else if (!key_start.empty() || !key_end.empty()) {
        // unload incomplete meta tablet
        if (!UnloadTabletSync(FLAGS_tera_master_meta_table_name, key_start, key_end,
                              meta.server_addr(), &status)) {
          TryKickTabletNode(meta.server_addr());
        }
      } else if (loaded) {
        // more than one meta tablets are loaded
        loaded_twice = true;
        if (!UnloadTabletSync(FLAGS_tera_master_meta_table_name, key_start, key_end,
                              meta.server_addr(), &status)) {
          TryKickTabletNode(meta.server_addr());
        }
        if (!UnloadTabletSync(FLAGS_tera_master_meta_table_name, "", "",
                              meta_tablet_meta.server_addr(), &status)) {
          TryKickTabletNode(meta.server_addr());
        }
      } else {
        loaded = true;
        meta_tablet_meta.CopyFrom(meta);
      }
    }
  }
  
  /*
   * 此处逻辑应该是有所改动，到此处要么对meta_tablet_addr赋值，要么直接返回false
   */
  std::string meta_tablet_addr;
  if (loaded && !loaded_twice) {
    meta_tablet_addr.assign(meta_tablet_meta.server_addr());
  } else if (!LoadMetaTablet(&meta_tablet_addr)) {
    return false;
  }
  // 从本地文件加载meta_table
  if (FLAGS_tera_master_meta_recovery_enabled) {
    const std::string &filename = FLAGS_tera_master_meta_recovery_file;
    while (!LoadMetaTableFromFile(filename)) {
      LOG(ERROR) << kSms << "fail to recovery meta table from backup";
      ThisThread::Sleep(60 * 1000);
    }
    // load MetaTablet, clear all data in MetaTablet and dump current memory
    // snapshot to MetaTable
    while (!tablet_manager_->ClearMetaTable(meta_tablet_addr) ||
           !tablet_manager_->DumpMetaTable(meta_tablet_addr)) {
      TryKickTabletNode(meta_tablet_addr);
      if (!LoadMetaTablet(&meta_tablet_addr)) {
        return false;
      }
    }
    TabletNodePtr meta_node = tabletnode_manager_->FindTabletNode(meta_tablet_addr, NULL);
    meta_tablet_ = tablet_manager_->AddMetaTablet(meta_node, zk_adapter_);
    LOG(INFO) << "recovery meta table from backup file success";
    return true;
  }

  // 向ts发送请求获取数据，然后加载其他的tablet
  StatusCode status = kTabletNodeOk;
  while (!LoadMetaTable(meta_tablet_addr, &status)) {
    TryKickTabletNode(meta_tablet_addr);
    if (!LoadMetaTablet(&meta_tablet_addr)) {
      return false;
    }
  }
  return true;
}
```

### LoadMetaTable

下面重点看一下向ts发送请求获取来加载tablet。

```C++
bool MasterImpl::LoadMetaTable(const std::string &meta_tablet_addr, StatusCode *ret_status) {
  tablet_manager_->ClearTableList();
  ScanTabletRequest request;
  ScanTabletResponse response;
  request.set_sequence_id(this_sequence_id_.Inc());
  request.set_table_name(FLAGS_tera_master_meta_table_name);
  request.set_start("");
  request.set_end("");
  TabletNodePtr meta_node = tabletnode_manager_->FindTabletNode(meta_tablet_addr, NULL);
  // 根据meta_node 创建meta_tablet_，并将meta_node 写入zk的/tera/root_table"
  meta_tablet_ = tablet_manager_->AddMetaTablet(meta_node, zk_adapter_);
  // 系统内通信的权限设置
  access_builder_->BuildInternalGroupRequest(&request);
  // 向ts发送请求获取meta_tablet的信息
  tabletnode::TabletNodeClient meta_node_client(thread_pool_.get(), meta_tablet_addr);
  while (meta_node_client.ScanTablet(&request, &response)) {
    if (response.status() != kTabletNodeOk) {
      SetStatusCode(response.status(), ret_status);
      LOG(ERROR) << "fail to load meta table: " << StatusCodeToString(response.status());
      tablet_manager_->ClearTableList();
      return false;
    }
    if (response.results().key_values_size() <= 0) {
      LOG(INFO) << "load meta table success";
      // 此处是加载成功之后的返回路径
      return true;
    }
    uint32_t record_size = response.results().key_values_size();
    LOG(INFO) << "load meta table: " << record_size << " records";

    std::string last_record_key;
    for (uint32_t i = 0; i < record_size; i++) {
      const KeyValuePair &record = response.results().key_values(i);
      last_record_key = record.key();
      char first_key_char = record.key()[0];
      if (first_key_char == '~') {
        user_manager_->LoadUserMeta(record.key(), record.value());
      } else if (first_key_char == '|') {
        if (record.key().length() < 2) {
          LOG(ERROR) << "multi tenancy meta key format wrong [key : " << record.key()
                     << ", value : " << record.value() << "]";
          continue;
        }
        char second_key_char = record.key()[1];
        if (second_key_char == '0') {
          /* The auth data stores in meta_table
           * |00User => Passwd, [role1, role2, ...]
           * |01role1 => [Permission1, Permission2, ...]
           */
          access_entry_->GetAccessUpdater().AddRecord(record.key(), record.value());
        } else if (second_key_char == '1') {
          // The quota data stores in meta_table
          // |10TableName => TableQuota (pb format)
          quota_entry_->AddRecord(record.key(), record.value());
        } else {
          LOG(ERROR) << "multi tenancy meta key format wrong [key : " << record.key()
                     << ", value : " << record.value() << "]";
          continue;
        }
      } else if (first_key_char == '@') {
        
        // 此处是重点，加载了meta_tablet的信息，后面会重点看一下加载了什么信息
        tablet_manager_->LoadTableMeta(record.key(), record.value());
      } else if (first_key_char > '@') {
        tablet_manager_->LoadTabletMeta(record.key(), record.value());
      } else {
        continue;
      }
    }
    std::string next_record_key = NextKey(last_record_key);
    request.set_start(next_record_key);
    request.set_end("");
    request.set_sequence_id(this_sequence_id_.Inc());
    response.Clear();
  }
  // 执行到此处表示加载失败
  SetStatusCode(kRPCError, ret_status);
  LOG(ERROR) << "fail to load meta table: " << StatusCodeToString(kRPCError);
  tablet_manager_->ClearTableList();
  return false;
}
```



```c++
void TabletManager::LoadTableMeta(const std::string& key, const std::string& value) {
  TableMeta meta;
  /* 
   * 从meta_tablet中读取了一条 TableMeta信息，既表的原信息
   * message TableMeta {
   *     optional string table_name = 1;
   *     optional TableStatus status = 2;
   *     optional TableSchema schema = 3;
   *     optional uint64 create_time = 5;
   *     optional AuthPolicyType auth_policy_type = 8;
   * }
   */
  ParseMetaTableKeyValue(key, value, &meta);
  TablePtr table = CreateTable(meta);
  StatusCode ret_status = kTabletNodeOk;
  if (meta.table_name() == FLAGS_tera_master_meta_table_name) {
    LOG(INFO) << "ignore meta table record in meta table";
    
    /*
     * AddTable 向TabletManager 成员变量all_tables_ 插入元素
     * 既插入了一个表名 -> 表对象的map 元素
     */
  } else if (!AddTable(table, &ret_status)) {
    LOG(ERROR) << "duplicate table in meta table: table=" << meta.table_name();
    // TODO: try correct invalid record
  } else {
    VLOG(5) << "load table record: " << table;
  }
}
```



### 总结

本部分的内容源码较多，但是逻辑简单，既先找到meta_tablet，然后读取其中的元数据（既table的元数据）进行加载。