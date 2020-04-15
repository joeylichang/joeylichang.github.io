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

![tera_master_state](../../../images/tera_master_state.png)

Master 的状态转换图如上所示，Master 以 kIsSecondary（从）身份启动，抢占 ZK 的 master_lock 成功之后通过事件 kGetMasterLock(event)  驱动转入 kOnRestore 状态，收集所有 TabletNode 节点的信息，获取 MetaTable 的位置，然后读取 MetaTable 的元数据进行加载，健在完成后进入 kIsReadonly 状态，然后尝试 kLeaveSafemode(event) （判断故障节点的 Tablet 是否超过总数的10%），如果成功则进入 kIsRunning 状态，开始 Master 的正常运行，以上都是在 Master 的初始化部分完成（后面介绍）。

Master 抢锁成功之后，如果没有 TabletNode（新集群未加入机器），会进入 kOnWait 状态等待机器的加入。Master 在 kIsRunning 期间，如果故障节点的 Tablet 占总数的10%及以上，会进入kIsReadonly 状态。



### Master Init

1. 向所有的 TabletNode 节点发送 Query 请求，目的是更新 TableNode 的内存信息 和 获取全部的 Tablets 信息。
   
   1. 根据 ZK 中 /tera/ts 子节点的 TabletNode 信息向所有的节点发送 Query 请求，通过信号量实现同步机制，保证所有的 Query 请求都收到回复 或者 重试失败 才继续向下进行。
   
   2. Query 返回的信息主要包括两部分，一部分是 TabletNode 信息，另一部分是 TableNode 负责的 Tablets 信息，收集的 TabletNode 更新内存中 TabletNodeManager 中相应的节点信息（TabletNode 在 Master 内的元数据详情见[TabletNode元数据解析](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/meta_data.md#tablenodes)）。
   
   3. 将所有的 TabletNode 的 Tablets 收集到一起之后放入Vector 供后续使用。
   
      **注意**：此处并没有更新 Tablet 在 Master 内存中的元数据，需要后面先加载元数据，因为元数据是要以 MetaTable 的为准（否则失去了 MetaTable 的意义），待加载完毕后与收集的信息进行对比，对于信息不一致有相应的逻辑。
   
2. 加载 MetaTable 中的元数据，在所有收集上来的 Tablets 中找到 MetaTablet 进行加载。

   1. 遍历上述收集的 Tablets 的 Vector，找到table_name == “meta_table” 的Tablet。

      注意：meta_table 加载的时候需要容错，有可能有多个 TableNode 上面有 MetaTable 的信息，可能是 MetaTable 有过搬迁但是没有 unload 成功等异常情况导致。

      1. MetaTablet 的 start_key 和 end_key 必须是 “”，既范围是正负无穷。
      2. 只有一个 MetaTablet ，如果在 Vector 中多以一个 Tablet 的 table_name 是 meta_table，也是异常情况。
      3. 针对上述异常情况，需要 unload 异常的 Tablet，如果 unload 失败，则 KickOff 节点。

   2. 如果没有找到 MetaTablet 或者有多于一个 MetaTablet 的情况，需要重新选择一个节点 load MetaTable（选取新节点的策略是 LoadBalance 中 根据容量策略进行选取的方式）。

   3. 加载 MetaTable 的元数据（也可以指定本地加载，既在本地磁盘读取文件进行加载）。

      1. Master 用 Client 的方式，向 MetaTable 发起 Scan 请求对其上的数据进行扫描。
      2. MetaTable 数据分为以下 5 种情况：
         1. key 的首字符 == '~'，表示 user 信息，LoadUserMeta。
         2. key 的首字符 == '|' 且 第二个字符是 '0'，表示是 Access 信息。
         3. key 的首字符 == '|' 且 第二个字符是 '1'，表示是 Quta 信息。
         4. key 的首字符 == '@'，表示是 Table 信息，LoadTableMeta。
         5. key 的首字符 > '@'，既table_name + ‘#’ + key_start，表示是 Tablet 信息LoadTabletMeta。
      3. Table 以及 Tablet 在Master 内存中的元数据信息详见[Tablet元数据解析](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/meta_data.md#tablet)

3. 加载 UserTable 的信息，根据收集上来的 Tablets 信息，对用户的 Tablets 进行处理。

   遍历第一步收集上来的 Tablets：

   1. 如果在 Master 内存中没有（既 MetaTable中没有），则 unload。

   2. 否则，绑定 Tablet 到其相应的 TabletNode（只是更新了 TabletNode 的size）。

   3. 如果 Tablet Status == kTableDisable，则将 Table 加入 disabled_tables 集合带后面统一处理。

   4. 如果 Tablet Status == kTabletUnloading || kTabletUnloading2，则进行move。

   5. 如果 Tablet Status == kTabletReady，则更新内存中 Tablet 的状态为kTabletReady。

      **注意** ：MetaTable 中存储的 Tablet 的状态都是 offline状态，在 load tablet 部分会详细介绍这其原因，其他状态都是 Master 内部维护的状态所以要根据 TabletNode 汇报的实际情况进行处理。Master 中 Tablet 的元数据 以 状态转换图见[Master中的Tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/tablet.md)。

      另外，还需要知道 TabletNode 存储的 Tablet状态有哪些，既kTabletNotInit、kTabletReady、kTabletOnLoad、kTabletUnloading、kTabletUnloading2，对于其他的状态 Master 不需要处理（比较好理解，不在此赘述）。

   此时，Master 内存中的 Tablet 的元数据 ，通过了与 TableNode 上的实际情况比对并进行了一些处理，还有一些 Tablets 可能存在于 MetaTable（既 Master 内存中）但是 TabletNode 上报中没有，下面要看一下这些 Tablets。遍历 Master 内存中所有的 Tablets：

   1. 跳过 MetaTablet 和 正在执行事务的 Tablets（上述正在load 或者 move的Tablet）。
   2. 跳过非 kTabletOffline 的 Tablet（如上所述，如果 tablet 来自TabletNode 的汇报状态一定不是MetaTable 中存储的kTabletOffline）。
   3. 如果 Tablet 所述的 Table 被 Disable 了，则将 Table 加入 disabled_tables 集合带后面统一处理。
   4. 如果 Tablet 所在的 TabletNode 在 Master 内存中，TryLoadTablet。
   5. 处理 Disable 的 Table，遍历所有的 Tablet 使其转换为 Disbale状态（[Tablet转换图]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/tablet.md#tablet%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2%E5%9B%BE](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/tablet.md#tablet状态转换图))），然后 unload tablet。

4. 重新加载 DeadNode 节点上面的 Tablets，重新选择一个节点进行加载（根据 LoadBalance中的 SizeScheduler 策略进行选取）。

   1. 遍历 Master 内存中所有的 Tablet。
   2. 选出处于 kTabletOffline 状态，且所在节点为非Ready状态的 Tablet，进行重新选取节点进行加载（选取策略根据 LB 的容量策略）。

##### Master 初始化源码解析见：

* [Master 初始化入口]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/init_master.md#master-%E5%88%9D%E5%A7%8B%E5%8C%96](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/init_master.md#master-初始化))
* [Master 收集meta_tablet信息](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/restore_ts.md)
* [Master 加载MeatTablet数据]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/restore_tablet.md#master-%E5%8A%A0%E8%BD%BDmeattablet%E6%95%B0%E6%8D%AE](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/restore_tablet.md#master-加载meattablet数据))
* [Master 加载UserTablet信息](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/restore_user_tablet.md)



### Master Period Task





#### Load Balance

#### Garbage Collection

### Procedure Arch

### Tabletnode State Machine

### Disaster Desgin

### AccessControl

### Quta

