# TabletNode Update

TN的update接口只接受来自master的更新schmea请求，其他的更新还不支持。

RemoteTabletNode层将任务交个线程池（10个线程）排队，在回调中调用TabletNodeImpl 的 Update家口没有什么复杂逻辑。



### TabletNodeImpl::Update

```c++
void TabletNodeImpl::Update(const UpdateRequest* request, UpdateResponse* response,
                            google::protobuf::Closure* done) {
  response->set_sequence_id(request->sequence_id());
  switch (request->type()) {
    // 只处理update schema类型
    case kUpdateSchema:
      LOG(INFO) << "[update] new schema:" << request->schema().DebugString();
      if (ApplySchema(request)) {
        LOG(INFO) << "[update] ok";
        response->set_status(kTabletNodeOk);
      } else {
        LOG(INFO) << "[update] failed";
        response->set_status(kInvalidArgument);
      }
      done->Run();
      break;
    default:
      LOG(INFO) << "[update] unknown cmd";
      response->set_status(kInvalidArgument);
      done->Run();
      break;
  }
}

bool TabletNodeImpl::ApplySchema(const UpdateRequest* request) {
  StatusCode status;
  io::TabletIO* tablet_io =
      tablet_manager_->GetTablet(request->tablet_name(), request->key_range().key_start(), request->key_range().key_end(), &status);
  if (tablet_io == NULL) {
    LOG(INFO) << "[update] tablet not found";
    return false;
  }
  // 更新tablet_io
  tablet_io->ApplySchema(request->schema());
  tablet_io->DecRef();
  return true;
}
```



### TabletIO::ApplySchema

```c++
void TabletIO::ApplySchema(const TableSchema& schema) {
  MutexLock lock(&schema_mutex_);
  // 成员变量切换
  SetSchema(schema);
  // cf lg关系映射，并设置相应TabletIO的成员变量，内部逻辑在load里面有介绍
  IndexingCfToLG();
  ldb_options_.compact_strategy_factory->SetArg(&schema);
}
```

