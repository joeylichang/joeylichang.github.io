# TableNodeServer Query

Query接口时接收Master的心跳探测，并且会返回当前节点 和 负责的tablet的信息。

### RemoteTabletNode::Query

```c++
void RemoteTabletNode::Query(google::protobuf::RpcController* controller,
                             const QueryRequest* request, QueryResponse* response,
                             google::protobuf::Closure* done) {
  uint64_t id = request->sequence_id();
  ThreadPool::Task callback =
      std::bind(&RemoteTabletNode::DoQuery, this, controller, request, response, done);
  // 10个线程的线程池，主要负责查询类操作（除read）
  lightweight_ctrl_thread_pool_->AddPriorityTask(callback);
}

void RemoteTabletNode::DoQuery(google::protobuf::RpcController* controller,
                               const QueryRequest* request, QueryResponse* response,
                               google::protobuf::Closure* done) {
  uint64_t id = request->sequence_id();
  int64_t start_micros = get_micros();
  LOG(INFO) << "run RPC (Query) id: " << id;
  access_entry_->GetAccessUpdater().UpdateTs(request, response);

  // Reset Quota iif version dismatch
  // 通过版本号进行校验更新quota，后续介绍quta
  quota_entry_->Update(request, response);

  tabletnode_impl_->Query(request, response, done);

  LOG(INFO) << "finish RPC (Query) id: " << id << ", cost " << (get_micros() - start_micros) / 1000
            << "ms.";
}
```



### TabletNodeImpl::Query

```c++
void TabletNodeImpl::Query(const QueryRequest* request, QueryResponse* response,
                           google::protobuf::Closure* done) {
  response->set_sequence_id(request->sequence_id());
  response->set_status(kTabletNodeOk);

  // sysinfo_ 统计信息将统计部分介绍
  TabletNodeInfo* ts_info = response->mutable_tabletnode_info();
  sysinfo_.GetTabletNodeInfo(ts_info);
  TabletMetaList* meta_list = response->mutable_tabletmeta_list();
  sysinfo_.GetTabletMetaList(meta_list);

  if (request->has_is_gc_query() && request->is_gc_query()) {
    std::vector<TabletInheritedFileInfo> inh_infos;
    /*
     * message TabletInheritedFileInfo {
     *     optional string table_name = 1;
     *     optional bytes key_start = 2;
     *     optional bytes key_end = 3;
     *     repeated LgInheritedLiveFiles lg_inh_files = 4;
     * }
     *
     * message LgInheritedLiveFiles {
     *     required uint32 lg_no = 1; 
     *     repeated uint64 file_number = 2;  // full file number, include tablet number
     * }
     * master 会根据这些信息对dfs的sst文件进行gc
     * 
     * GetInheritedLiveFiles 逻辑：
     * 1. 遍历所有的TabletIO，调用其AddInheritedLiveFiles，根据返回值回填上述pb结构信息
     * 2. TabletIO 调用 leveldb::DBTable接口AddInheritedLiveFiles，DBTable遍历lg调用leveldb::DBIpml的相应结构，期间逻辑较简单，不赘述。
     * 3. DBIpml 内部 根据leveldb::version维护的正在使用的sst文件信息进行返回，并校验，校验逻辑是文件名字会带有tablet的编号。
     */
    GetInheritedLiveFiles(&inh_infos);
    for (size_t i = 0; i < inh_infos.size(); i++) {
      TabletInheritedFileInfo* inh_info = response->add_tablet_inh_file_infos();
      inh_info->CopyFrom(inh_infos[i]);
    }

    // only for compatible with old master
    std::vector<InheritedLiveFiles> inherited;
    GetInheritedLiveFiles(inherited);
    for (size_t i = 0; i < inherited.size(); ++i) {
      InheritedLiveFiles* files = response->add_inh_live_files();
      *files = inherited[i];
    }
  }

  // if have background errors, package into 'response' and return to 'master'
  std::vector<TabletBackgroundErrorInfo> background_errors;
  // 见下面介绍，主要是对加载失败的tablet强行卸载，还有就是获取一下压缩的错误
  GetBackgroundErrors(&background_errors);
  for (auto background_error : background_errors) {
    TabletBackgroundErrorInfo* tablet_background_error = response->add_tablet_background_errors();
    tablet_background_error->CopyFrom(background_error);
  }
  done->Run();
}


void TabletNodeImpl::GetBackgroundErrors(
    std::vector<TabletBackgroundErrorInfo>* background_errors) {
  std::vector<io::TabletIO*> tablet_ios;
  tablet_manager_->GetAllTablets(&tablet_ios);
  std::vector<io::TabletIO*>::iterator it = tablet_ios.begin();
  uint64_t reported_error_msg_len = 0;
  // 遍历所有的tablet
  while (it != tablet_ios.end()) {
    io::TabletIO* tablet_io = *it;
    // 加载过程中遇到的错误，进行下载，主要是目录权限引起的问题
    if (tablet_io->ShouldForceUnloadOnError()) {
      StatusCode status;
      if (!tablet_io->Unload(&status)) {
      }
      if (!tablet_manager_->RemoveTablet(tablet_io->GetTableName(), tablet_io->GetStartKey(), tablet_io->GetEndKey(), &status)) {
      }
      tablet_io->DecRef();
      it = tablet_ios.erase(it);
      continue;
    }
    std::string background_error_msg = "";
    // 底层调用leveldb的GetProperty，关注返回的信息中compaction_error 
    tablet_io->CheckBackgroundError(&background_error_msg);
    if (!background_error_msg.empty()) {
      std::string msg = tera::sdk::StatTable::SerializeCorrupt(
          sdk::CorruptPhase::kCompacting, local_addr_, tablet_io->GetTablePath(), "",
          background_error_msg);

      reported_error_msg_len += msg.length();

      // if the length of error message overrun the limit，
      // only part of them would be reported
      if (reported_error_msg_len < kReportErrorSize) {
        tera::TabletBackgroundErrorInfo background_error;
        background_error.set_tablet_name(tablet_io->GetTablePath());
        background_error.set_detail_info(msg);
        background_errors->push_back(background_error);
      }
    }
    ++it;
    tablet_io->DecRef();
  }
}
```

