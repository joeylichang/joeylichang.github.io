# Master 通用流程处理架构

Master对CreateTable、分裂、合并、迁移等12种操作进行了抽象，使用同一的架构进行处理，本部分对抽象的通用架构进行分析。Procedure的类图，如下图所示：

<img src="../../../../../images/tera_master_producre_arch.png" alt="tera_master_producre_arch" style="zoom:30%;" />

### 原理

1. 在MasterImpl内部包含ProcedureExecutor的成员变量，涉及上述12种操作时调用ProcedureExecutor的AddProcedure将任务加到队列里。
2. ProcedureExecutor后台调度队列中的任务（使用信号量），从队列中取出ProcedureWrapper，ProcedureWrapper是对Procedure进行了一层封装，内部有同步和限流机制，既针对10中操作并行能够执行最多的任务数量进行了限制。
3. ProcedureLimiter实现了同步和限流机制，支队Merge、Spilt、Move、Load进行了限制。
4. 12中子流程均集成自Procedure父类，ProcedureExecutor调用子任务的RunNextStage开始执行各自的具体逻辑。
5. 每种子流程又会分成若干段运行，ProcedureExecutor遍历队列执行任务时，知会执行子流程中的一段子任务，这样保证后加入的子流程不会被饿死。
6. 注意：子流程中子任务之间需要维持状态，ProcedureExecutor不负责，而是交给子流程去管理，ProcedureExecutor在每次调用RunNextStage会调用Done检查子流程是否结束。

### 源码

ProcedureExecutor后台调度逻辑：

```c++
void ProcedureExecutor::ScheduleProcedures() {
  while (running_) {
    std::map<uint64_t, std::shared_ptr<ProcedureWrapper>> procedures;
    {
      std::unique_lock<std::mutex> lock(mutex_);
      while (procedures_.empty() && running_) {
        // 等待子流程的加入
        cv_.wait(lock);
      }
      procedures = procedures_;
    }

    // 遍历队列调用procedures的RunNextStage
    for (auto it = procedures.begin(); it != procedures.end(); ++it) {
      auto proc = it->second;
      const std::string proc_id = proc->ProcId();
      // TrySchedule 调用 ProcedureLimiter 的 GetLock，里面有限流策略
      if (proc->TrySchedule()) {
        ThreadPool::Task task =
            std::bind(&ProcedureWrapper::RunNextStage, proc, shared_from_this());
        thread_pool_->AddTask(task);
      }
    }
    // sleep 10ms before start next schedule round
    usleep(10 * 1000);
  }
}
```



接下来看一下ProcedureLimiter的限流策略。

```C++
ProcedureLimiter() {
    // Merge 并发上线 10
    SetLockLimit(LockType::kMerge, static_cast<uint32_t>(FLAGS_master_merge_procedure_limit));
    // Split 并发上线 10
    SetLockLimit(LockType::kSplit, static_cast<uint32_t>(FLAGS_master_split_procedure_limit));
    // Move 并发上线 100
    SetLockLimit(LockType::kMove, static_cast<uint32_t>(FLAGS_master_move_procedure_limit));
    // Load 并发上线 300
    SetLockLimit(LockType::kLoad, static_cast<uint32_t>(FLAGS_master_load_procedure_limit));
    // Unload 并发上线 100
    SetLockLimit(LockType::kUnload, static_cast<uint32_t>(FLAGS_master_unload_procedure_limit));
    {
      std::lock_guard<std::mutex> guard(mutex_);
      in_use_[LockType::kMerge] = 0;
      in_use_[LockType::kSplit] = 0;
      in_use_[LockType::kMove] = 0;
      in_use_[LockType::kLoad] = 0;
      in_use_[LockType::kUnload] = 0;
    }
  }

bool GetLock(const LockType& type) {
    if (type == LockType::kNoLimit) {
      VLOG(20) << "[ProcedureLimiter] get lock for type:" << type << " success";
      return true;
    }
    assert(limit_.find(type) != limit_.end());
    assert(in_use_.find(type) != in_use_.end());

    std::lock_guard<std::mutex> guard(mutex_);
    if (in_use_[type] >= limit_[type]) {
      VLOG(20) << "[ProcedureLimiter] get lock for type:" << type
               << " fail, reason: lock exhaust, lock limit:" << limit_[type]
               << ", in use:" << in_use_[type];
      return false;
    }
    ++in_use_[type];
    VLOG(20) << "[ProcedureLimiter] get lock for type:" << type
             << " success, lock limit:" << limit_[type] << ", in use:" << in_use_[type];
    return true;
  }
```



ProcedureWrapper调度子流程的子任务逻辑。

```c++
void ProcedureWrapper::RunNextStage(std::shared_ptr<ProcedureExecutor> proc_executor) {
  // 调用子流程的子任务
  proc_->RunNextStage();
  scheduling_.store(false);
  // 判断是否完成
  if (Done()) {
    // 释放限流资源
    ProcedureLimiter::Instance().ReleaseLock(proc_->GetLockType());
    VLOG(23) << "procedure executor remove procedure: " << ProcId();
    proc_executor->RemoveProcedure(ProcId());
  }
}
```



