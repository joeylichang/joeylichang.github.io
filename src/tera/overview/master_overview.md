* **Navigation**
  * [Master Arch](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#master-arch)
  * [Master State Machine](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#master-state-machine)
  * [Master Init](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#master-init)
  * [Master Period Task](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#master-period-task)
    * [Load Balance](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#load-balance)
    * [Garbage Collection](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#garbage-collection)
  * [Procedure Arch](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#procedure-arch)
  * [Tabletnode State Machine](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#tabletnode-state-machine)
  * [Disaster Desgin](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#disaster-desgin)
  * [AccessControl](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#access-control)
  * [Quta](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#quta)

### Master Arch

#### Mater Class arch

<img src="../../../images/tera_master_arch.png" alt="tera_master_arch" style="zoom:100%;" />

Master 核心的类架构图如上所示，上述类图过于详细繁杂，下面分几大块进行介绍（后续针对各自逻辑会详细介绍）：

1. MasterImpl：Master的核心模块，也是所有功能模块初始化 和 入口的地方。在MasterEntry模块中对其进行初始化，MasterEntry 除了 MasterImpl 还有 RPC 模块的初始化，Master的核心逻辑都集中在MasterImpl。
2. TabletNodeManager：负责TabletNode管理的模块，一个 TabletNodeServer 映射为 一个TabletNode。
3. TabletManager：Tablet的管理模块，内部通过 map<table_name, Table> 映射管理所有的Table，每一个 Table 通过TabletList 管理其 Tablet。Tablet 内部有addr（所在的节点地址）与 TabletNode 进行关联。
4. ProcedureExecutor：一套任务调度的框架（后面会详细介绍），Table 的 create、disable、enable、delete、update，以及 Tablet 的 load、unload、move、spilt、merge等流程都继承自ProcedureExecutor 使用统一的调度模式。
5. Scheduler：负载均衡的框架，支持按大小（SizeScheduler）、流量（LoadScheduler）两种模式进行负载均衡（后面会详细介绍）。
6. Access*：权限相关模块（后续会详细介绍），Tera 在开源的时候阉割了百度内部使用的权限管理系统。
7. MasterQuotaEntry：Quta相关模块（后续会详细介绍）。
8. AbnormalNodeMgr：防止因网络抖动导致 TabletNode 频繁加入离开的容灾模块（后面会详细介绍）。
9. UserManager：用户管理模块。



#### Master Thread arch

有了上述模块的角色了解之后，下面看一下 Master 的线程模型。

```c++
|_MasterEntry
  |_RemoteMaster                ThreadPool_A（10）
  |_RemoteMultiTenancyService   ThreadPool_A（10）
  		|
  		|_MasterImpl                ThreadPool_B（20）
  		|_TabletManager             ThreadPool_B（20）
  				|
  				|_query_thread_pool_      ThreadPool_C（20）
```

Master 主要有三个线程池，负责 响应 Table、Tablet 的 RPC（RemoteMaster）服务 以及 响应 权限、Quta的 RPC（RemoteMultiTenancyService服务）共用一个线程池（默认10个线程）。

MasterImpl 和 TabletManager 公用一个线程池，MasterImpl主要是一些周期性任务 和 向TabletNode 发送请求等 、 TabletManager 会向 TabletNode 发送请求、各种Procedure（Table 创建、Tablet搬迁等刘海成） 会使用多线程完成，他们共一个线程池（默认20个线程）。

MasterImpl 内部 还有一个线程池 query_thread_pool_ 专门用于心跳探测，默认20个线程。加上主线程 Master模块默认使用51个线程。



### Master State Machine





### Master Init

### Master Period Task

#### Load Balance

#### Garbage Collection

### Procedure Arch

### Tabletnode State Machine

### Disaster Desgin

### AccessControl

### Quta

