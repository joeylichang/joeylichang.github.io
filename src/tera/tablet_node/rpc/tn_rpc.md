

# TabletNode RPC

### RemoteTabletNode

RemoteTabletNode负责TabletNode RPC的远程调用逻辑。先看一下其重要的成员变量：

```c++
TabletNodeImpl* tabletnode_impl_;     // rpc层做一些校验限流逻辑之后会调用其相应接口完成操作
scoped_ptr<ThreadPool> ctrl_thread_pool_;  // loadtablet、unloadtablet 在任务较多是使用这个线程池，默认20个线程
scoped_ptr<ThreadPool> lightweight_ctrl_thread_pool_;  // loadtablet、unloadtablet 任务不重 或者 query、CmdCtrl、computespilt、update 使用的线程池 默认 10个线程
scoped_ptr<ThreadPool> write_thread_pool_;  // 写请求的线程池 默认10个线程
scoped_ptr<ThreadPool> read_thread_pool_;   // 读请求的线程池 默认40个线程
scoped_ptr<ThreadPool> scan_thread_pool_;   // scan请求的线程池 默认30个线程
scoped_ptr<ThreadPool> compact_thread_pool_;// 压缩请求的线程池 默认30个线程
scoped_ptr<RpcSchedule> read_rpc_schedule_; // read rpc的调度侧率（一种排队算法，见后面介绍）
scoped_ptr<RpcSchedule> scan_rpc_schedule_; // scan rpc的调度侧率（一种排队算法，见后面介绍）
scoped_ptr<RpcSchedule> quota_retry_rpc_schedule_; // quota rpc的调度侧率（一种排队算法，见后面介绍）

enum TabletCtrlStatus {
  kCtrlWaitLoad = kTabletWaitLoad,
  kCtrlOnLoad = kTabletOnLoad,
  kCtrlWaitUnload = kTabletWaitUnload,
  kCtrlUnloading = kTabletUnloading,
};

std::mutex tablets_ctrl_mutex_;   // 未使用
std::map<std::string, TabletCtrlStatus> tablets_ctrl_status_;  // path -> table load/unload状态 用于选择lightweight_ctrl_thread_pool_ 或者 ctrl_thread_pool_ 使用

std::unique_ptr<auth::AccessEntry> access_entry_;  // 权限校验
std::shared_ptr<quota::QuotaEntry> quota_entry_;   // quta模块
```

### RPC的调度策略

![tera_tn_rpc_arch](../../../../images/tera_tn_rpc_arch.png)

TableNodeServer 的读、Scan、Quta请求的调度策略都是相同的，类图及重要的类成员变量如上图所示。

整个调度的策略，在RPC远程调用做一下队列的缓冲。

RpcSchedule 内部维护 表名 -> 调度单元的映射（既std::map<TableName, ScheduleEntity*> TableList），调度单元（ScheduleEntity）内部有有一个队列（既TaskQueue）记录RPC任务（既RpcTask）。

RpcSchedule 维护三个重要接口EnqueueRpc、DequeueRpc、FinishRpc。

1. EnqueueRpc：通过tablename 获取调度单元，再获取对应的队列，将RpcTask加入队列。如果TaskQueue的pending_count不为0，则调用FairSchedulePolicy的Enable的接口，将调度单元设置为可调度。

2. DequeueRpc：调用FairSchedulePolicy的pick接口获取要执行的RpcTask。Pick的策略是遍历std::map<TableName, ScheduleEntity*> TableList，判断当前调度单元（ScheduleEntity）是否是enable，如果是取出队列，从头部取出一个RpcTask去执行。如果调度单元的pending_count为0，则调用FairSchedulePolicy的Disable接口将调度单元（ScheduleEntity）设置为不可调度

   ##### 注意：如果一个table积压的任务较多，其他table的任务可能长时间等待，因超时饿死。

3. FinishRpc：调用FairSchedulePolicy的Done的接口，更新调度单元（ScheduleEntity）的统计数据。

##### 注意：整个调度策略为了不同维度的统计数据，主要包括一下

1. RpcSchedule： pending_task_count_、running_task_count_（整个调度实例的运行 和 挂起的任务数量，read、scan、quota是三个调度实例）
2. TaskQueue：一个表对应一个TaskQueue，既表维度的pending_count、running_count。
3. FairScheduleEntity：调度单元，也是表维度的pickable（是否可调度，取决于TaskQueue的pending_count是否大于0）、last_update_time（调度单元最后一次更新时间）、elapse_time（调度单元总耗时）、running_count（调度单元总运行rpc数量）