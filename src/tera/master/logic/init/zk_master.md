# Master-ZK 交互逻辑

### Master 状态机

master 通过zk实现抢主逻辑，在内部维持一个状态机，抢主成功之后将从初始化的从变为Restore状态，去加载TableNode，Mater状态机的流程如下：

![tera_master_state](../../../../images/tera_master_state.png)

### Master 初始化时与ZK交互的节点

```protobuf
/tera
     /master-lock
     /master                   // 主master，同步锁，如果是从的话会一直在这里抢锁
     /root_table               // metatable所在的节点地址
     /safemode                 // 是否处于安全模式
     /ts
        /0000000001(ts_addr)
        /0000000002(ts_addr)    // 版本号 - tablenode地址，取版本号大的为准
```

zk内部所有的操作，拥有相同的配置：

```protobuf
tera_zk_timeout           // 超时时间 10s
tera_zk_retry_period      // 重试间隔 3s
tera_zk_retry_max_times   // 重试次数 10次，所以整体超时时间 11 * 13s = 143s > 2min
```

### MasterZkAdapter 主要逻辑

MasterZkAdapter 在MasterImpl 的初始化函数中进行构造和初始化，主要逻辑如下：

```c++
bool MasterZkAdapter::Init(std::string* root_tablet_addr,
                           std::map<std::string, std::string>* tabletnode_list, bool* safe_mode) {
  MutexLock lock(&mutex_);

  if (!Setup()) {         // zk 的初始化函数，在这里面初始化了根目录 "/tera"
    return false;         // 注意：初始化的时候进行了watch的回调设置
  }

  if (!LockMasterLock()) {  // 同步抢占锁，如果抢占失败一直卡在这里，内部是信号量 + 锁实现
    Reset(); 	              // zk 清理工作
    return false;
  }

  if (!WatchMasterLock()) { // 注意这里的的watch相当于check是否抢占了锁，并没有watch的功能
    UnlockMasterLock();     // zk的watch在Setup里面进行了
    Reset();
    return false;
  }

  if (!CreateMasterNode()) { // 创建"/tera/master"节点
    UnlockMasterLock();
    Reset();
    return false;
  }

  bool root_tablet_node_exist = false;   // 查看"/tera/root_table"是否存在
  if (!WatchRootTabletNode(&root_tablet_node_exist, root_tablet_addr)) {
    DeleteMasterNode();
    UnlockMasterLock();
    Reset();
    return false;
  }

  if (!WatchSafeModeMark(safe_mode)) {    // 查看"/tera/"是否存在
    DeleteMasterNode();
    UnlockMasterLock();
    Reset();
    return false;
  }

  if (!WatchTabletNodeList(tabletnode_list)) { // 获取"/tera/ts"子节点数据
    DeleteMasterNode();
    UnlockMasterLock();
    Reset();
    return false;
  }

  return true;
}
```

整体流程比较简单清晰，值得注意的是SetUp里面对zk初始化时对节点创建、删除、数据变更进行了会掉设置，用于初始化的Adapter都会实现相应的回调，在回调里面处理关心的事件，在Master与Zk交互的过程中只关心节点删除、子节点变更（ts目录的子节点）。

```c++
/*
 * ts 节点被删除，标记为safe_mode状态
 * 禁止读操作
 */
void MasterZkAdapter::OnTabletNodeListDeleted() {
  LOG(ERROR) << "ts dir node is deleted";
  if (!MarkSafeMode()) {
    // zookeeper node is not allowed to be deleted unless it has no child node,
    // thus once
    // ts dir is deleted, master must be in kOnWait status
    master_impl_->DisableQueryTabletNodeTimer();
    DeleteMasterNode();
    UnlockMasterLock();
    Reset();
  }
}

/*
 * root_table节点被删除，重写
 */
void MasterZkAdapter::OnRootTabletNodeDeleted() {
  LOG(ERROR) << "root tablet node is deleted";
  std::string root_tablet_addr;
  if (master_impl_->GetMetaTabletAddr(&root_tablet_addr)) {
    if (!UpdateRootTabletNode(root_tablet_addr)) {
      // let master behavoir as it lost MasterLock
      master_impl_->DoStateTransition(MasterEvent::kLostMasterLock);
      master_impl_->DisableQueryTabletNodeTimer();
      DeleteMasterNode();
      UnlockMasterLock();
      Reset();
    }
  } else {
    LOG(ERROR) << "root tablet not loaded, will not update zk";
  }
}

/*
 * 通过状态机，降为从节点
 */
void MasterZkAdapter::OnMasterNodeDeleted() {
  LOG(ERROR) << "master node deleted";
  master_impl_->DoStateTransition(MasterEvent::kLostMasterLock);
  master_impl_->DisableQueryTabletNodeTimer();
  UnlockMasterLock();
  Reset();
}

/*
 * 锁节点被删除，禁止读，并退出程序
 */
void MasterZkAdapter::OnZkLockDeleted() {
  LOG(ERROR) << "master lock deleted, kill-self";
  master_impl_->DisableQueryTabletNodeTimer();
  Reset();
  _Exit(EXIT_FAILURE);
}
```



