* **Navigation**
  * [Table Procedure](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#table-procedure)
    * [create table](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#create-table)
    * [disable table](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#disable-table)
    * [enable table](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#enable-table)
    * [delete table](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#delete-table)
    * [update table](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#update-table)
  * [Tablet Procedure](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#tablet-procedure)
    * [load tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#load-tablet)
    * [unload tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#unload-tablet)
    * [move tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#move-tablet)
    * [spilt tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#spilt-tablet)
    * [merge tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#merge-tablet)
  * [Client Procedure](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#client-procedure)
    * [update(write/delete)](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#updatewritedelete)
    * [read](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#read)
    * [scan](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#scan)
    * [transaction](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#transaction)
      * [single row transaction](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#single-row-transaction)
      * [global transaction](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md#global-transaction)
      

### Table procedure

#### create table

* Client

  Client 侧主要的数据结构：[TableDescImpl](https://github.com/joeylichang/tera/blob/master/src/sdk/schema_impl.h)、[LGDescImpl](https://github.com/joeylichang/tera/blob/master/src/sdk/schema_impl.h)、[CFDescImpl](https://github.com/joeylichang/tera/blob/master/src/sdk/schema_impl.h)，负责 Table、LG、CF 的 Schema 的设定，然后转换成 Request 发送给 Master。

  ```protobuf
  message CreateTableRequest {
      required uint64 sequence_id = 1;
      required string table_name = 2;    // 表名
      optional TableSchema schema = 3;   // TableDescImpl 转换的 Schema，内部包含LGDescImpl、CFDescImpl
      repeated bytes delimiters = 6;     // Tablet 水平切分的 RowKey 范围
      optional bytes user_token = 7;
      optional IdentityInfo identity_info = 8;
  }
  ```

  

* Master

  Master 侧 RPC 回调层收到请求后，会创建任务加入线程池（默认10个线程，详情见[Master Thread arch](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#master-thread-arch)），在 MasterImpl 层创建 CreateTableProcedure（继承自 [Procedure](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md#procedure-arch)） 完成建表流程。线面看一下 Master 内部建表的主要步骤：

  1. Prepare
     1. 检查是否已经存在相同表名的表、校验用户是否是 RootUser、移除 DFS 同名目录。
     2. 校验 RowKey 拆分的是否正确（Tablet 水平切分的 RowKey 是否有重合）、校验是否超过了 Tablet 预分配的数量（默认 10w）、校验是否有 lg（至少一个）。
     3. 在 Master 内存中，创建 Tablet 和 Tablet。
  2. UpdateMeta
     1. 异步更新 [Table、Tablet 的元数据到](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#metatablet) MetaTable。
     2. 更新失败直接跳转步骤 4。
  3. LoadTablets：遍历 Table 创建的 Tablet，为每一个 Tablet 创建 LoadTabletProcedure（后面介绍），进行 Tablet 的加载。
  4. Eof：结束 RPC 请求。

* TabletNode

  TabletNode 侧针对 CreateTable ，接收到的是 LoadTablet 请求（后面详细介绍）。



##### **注意**

1. CreateTable 返回之后，LoadTablet 未必结束，此时表仍然不可用。

2. 先更新内存，然后异步更新 MetaTable（默认重试 5次），如果更新 MetaTable 失败，怎么处理？

   内存中已经有相应的元数据，只能走 Disable->Delete 流程删除 Table，然后重新创建，直接创建会失败。

3. 内存、MetaTable 更新成功，但是load tablet失败 怎么处理？

   对于 CreateTable 在内存和 MetaTable 中更新成功，则认为创建成功，至于 Tablet 是否加载成功，在LoadTablet 和 心跳探测部分有机制保证 Tablet 的正常。

4. CreateTable 成功之后，待 Tablet 加载完毕，即可使用，为什么还需要 EnableTable 操作？

   EnableTable 是与 DisableTable 对应的，在删除之前必须 DisableTable 防止误删、以及表暂停、恢复使用。



##### 源码解析

[Master Create Table 流程]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/table/creat_table_procedure.md#master-create-table-%E6%B5%81%E7%A8%8B](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/table/creat_table_procedure.md#master-create-table-流程))



#### disable table

* Table 状态机转换图

![tera_master_table_state_machine](../../../images/tera_master_table_state_machine.png)

从 Table 状态转换图可知， Table 被删除之前，必须进入 Disable 状态，在 Disable 状态可以转化为 Enable（创建 Table 之后默认为 Enable 状态），所以 Disable 是一个删除之前的暂存状态，方便 Table 重新上线，也是 Delete Table 的一个缓冲状态。

* Master
  1. Prepare：权限校验，RootUser 或者 Schema 指定了的 Admin 用户。
  2. DisableTable：Table 状态机转化为 TableDisable
  3. UpdateMeta：只更新 Table 的元数据，既 TableMeta 中表的状态为 TableDisable，其他部分没变。
  4. DisableTablets：遍历 Table 中所有的 Tablet，为每一个 Tablet 创建一个 UnloadTabletProcedure（后续详细介绍） 进行卸载，值得注意的是，有同步机制保证所有的 Tablet 都卸载成功之后才会进入 Eof 阶段。
  5. Eof：结束 RPC 请求。

* TableNode：在 Disable Table 流程中，TabletNode 主要接受的请求是 UnloadTablet（后续会详细介绍）。



##### 注意

1. Disable Table 成功返回，表既不可用（与 Create Table 不同），有同步及机制保证所有的 UnloadTablet 完成再返回。

2. Disable Table  先更新内存（状态机），然后异步更新 MetaTable（默认重试 5次），如果更新 MetaTable 失败，怎么处理？

   需要先 Enable 再 Disable。

3. 如果 UnloadTablet 很慢，Disable Table 请求超时怎么办？

   UnloadTablet 正常应该是ms 级完成的操作，在一些异常情况下会有延迟。但是一旦 Master 开始 UnloadTablet 就一定会不断的重试直到所有的 Tablet Unload 成功。



##### 源码解析

[Master Disable Table 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/table/disable_table_procedure.md)



#### enable table

* Master
  1. Prepare：权限校验，RootUser 或者 Schema 指定了的 Admin 用户。
  2. EnableTable：Table 状态机转化为 TableEnable
  3. UpdateMeta：只更新 Table 的元数据，既 TableMeta 中表的状态为 TableEnable，其他部分没变。
  4. EnableTablets：遍历 Table 中所有的 Tablet，为每一个 Tablet 创建一个 LoadTabletProcedure（后续详细介绍） 进行卸载。值得注意的是，此处并没有同步机制保证所有的 Tablet 都加载成功之后载进入 Eof 阶段（与 Disable 不同），与 Create Table 流程相同，既表 Enable 成功返回了，但是表会短时不可用。
  5. Eof：结束 RPC 请求。
* TableNode在 Enable Table 流程中，TabletNode 主要接受的请求是  LoadTablet（后续会详细介绍）。



##### 注意

1. Enable Table 请求返回，但是 Tablet 可能没有 Load 完成，表会存在短时不可用。

2. Enable Table  先更新内存（状态机），然后异步更新 MetaTable（默认重试 5次），如果更新 MetaTable 失败，怎么处理？

   先Disable 再重新 Enable。

3. LoadTablet 如果又失败怎么办？

   通过心跳 和 Load Balance 会对Tablet 进行迁移，保障 Tablet 可以正常提供服务。



##### 源码解析

[Master Enable Table 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/table/enable_table_procedure.md)



#### delete table

* Master
  1. Prepare：权限校验，RootUser 或者 Schema 指定了的 Admin 用户。
  2. DeleteTable：从 DisableTable 状态转换为 DeletingTable。删除table 的 quta， quta 都是table级别的。
  3. UpdateMeta：只更新 Table 的元数据，既 TableMeta 中表的状态为 DeletingTable，其他部分没变。
  4. Eof：结束 RPC 请求。
* TabletNode：Disable 过程 TabletNode 已经 UnloadTablets，Delete 流程只是修改元数据，TabletNode 没有任何工作。



##### 注意

1. Delete Table  先更新内存（状态机），然后异步更新 MetaTable（默认重试 5次），如果更新 MetaTable 失败，怎么处理？

根据状态转化图可以，需要先 Disable Table，是 Table 状态转化为 Disable 之后重新 Delete。



##### 源码解析

[Master Delete Table 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/table/delete_table_procedure.md)



#### update table

* Master

  1. Prepare

     1. 权限校验，RootUser 或者 Schema 指定了的 Admin 用户。
     2. 更新的 Schema 必须包含至少一个 lg。
     3. 双 buffer 更新 Table 内存中的 Schema，先更新备 Buffer。
     4. 判断 Table 必须是 Enable 状态，并且 Schema 的 CF 有变化（Schema 可以不更新CF，如果不更新不涉及到 Tablet 的变更，则没有此步骤）
        1. 遍历 Table 所有的 Tablet，抢占其 事务锁（一个 Tablet 一个）
        2. Tablet 状态必须是 kTabletReady 或者 kTabletOffline
        3. 上述两个条件有一个 Tablet 不满足，直接跳转步骤 4。

  2. UpdateMeta

     1. 异步更新 MetaTable 相应 Table 的 TableMeta。
     2. 如果Table 是 Enable 状态，并且 IsUpdateCf(table_)，跳转步骤 3。
     3. 否则切换 Master 内存中备份的 Schema 到 正式的 Buffer 中，跳转步骤 4。

  3. TabletsSchemaSyncing

     1. 遍历 Table 所有的 Tablet，对齐发送 Update 请求更新 Schema。
     2. 超过重试次数仍然失败的请求，会 kick_off 节点（Tablet 会在后面统计重新 Load）。
     3. 上述步骤有同步机制保证所有 Ready 的 Tablet 都会被更新完才能继续。
     4. 对于 Offline 的 Tablet，在处理完 Ready 的 Tablet 之后，统一一起 Load，因为此时再Load 就是使用最新的 Schema 了，无需关注 Update 逻辑。
     5. 切换 Master 内存中备份的 Schema 到 正式的 Buffer 中。

     **注意**：在 Prepare 阶段，所有的 Tablet 都必须是 Ready 和 Offline，否则请求直接返回失败。在 TabletsSchemaSyncing 阶段期间是不会 Tablet 状态的变更，因为所有的 Tablet 都上来事务锁，这个流程没有结束是不会有变更的。

  4. Eof：释放 Table 的事务锁，并结束 RPC 请求。

* TabletNode
  1. 根据请求中的 TableName、KeyRange 找到对应的 TabletIo。
  2. 更新 TabletIo 层的 Schema，以及 LG 和 CF 的映射关系（Scan 等操作会用到）。
  3. 压缩策略里面的 Schema 成员变量也进行更新。



##### 注意

1. Schema 更新除了更新 CF、LG 也可以更新其他的配置，但是需要谨慎不是所有的都能更新（比如 是否支持 Rowkey Hash等），不涉及到CF 和 LG 是不会与 TableNode 进行交互的。

2. 先更新内存备份，再更新 MetaTable（有必要的话同步更新 TableNode），然后切换内存的备份为正式。这个流程与之前的有所不同，主要就是 Schema 是在线更新，既更新的过程中 Table 仍然可用。

   1. 更新内存备份成功，更新 MetaTable 失败，会清除内存备份，使用旧的 Schema ，结束流程
   2. 同步更新 TableNode 有失败，会 KickOff 节点，完成 Ready 的 Tablet 之后，统一用最新的Schema 去 Load Tablet。

3. 之前 Table 的流程中，都是先更新内存，然后更新 MetaTable，一旦更新 MetaTable 失败，需要手动回滚。Update 的不同之处是，先更新内存备份，然后更新 MetaTable，最后切换内存，每一步的失败都在自身进行了回滚。这么做的原因也是 Update 的特殊性，因为更新期间 Table 必须保证可用。

4. Update 处理的流程中的 Tablet 必须是 Ready 或者 Offline 状态，如果 Update 期间，其状态有变化怎么办？

   由于 Table 和 Tablet 都加了事务锁，所以期间不会有流程去改变其状态。

5. 笔者理解只接收qua、col、lg的增减，不支持qua换cf 和 cf 换lg。因为 LevelDB 底层已经根据 LG 对数据进行了划分，并且 LG 内部已经根据 CF 拍好了序列。如果发生了转换，旧数据相当于抛弃不要了，没有数据转换的逻辑。



##### 源码解析

[Master Update Table 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/table/update_table_procedure.md)

[TabletNode Update](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/procedure/tn_update.md)



### Tablet procedure

##### Tablet 状态转化图

![tera_tablet_state_change](../../../images/tera_tablet_state_change.png)

#### load tablet

* Master

  1. 初始tablet 状态一定是 offline（ <- loadfail） 和 delayoffline
  2. 如果des 为空，根据size策略选取一个节点                                     -> tablet 进入 offline
  3. 如果des down（节点为offline）：
     1. 如果是meta表，根据size 选节点，然后绑定                    -> tablet 进入 offline
     2. 非meta表，如果des 还在内存中，绑定tablet 到 node   -> tablet 进入 offline
     3. 非meta表，如果des 不在内存，且为pendingoff状态     -> tablet 进入 offline
     4. 以上都不是 ，根据size 选节点，然后绑定                         -> tablet 进入 offline
  4. offline 状态会跳转到第一步 重新选取节点继续往下，截止到这里 tablet 处于 offline 状态
  5. 如果 des 正常：
     1. 如果是meta表，load tablet                                                 -> tablet 进入 loading
     2. 非meta表，更新meta （异步更新，认为一定成功），这里会无限重试  -> 更新 MetaTable
     3. 非meta表，目标节点不处于busy状态（Load计数闸值），load tablet   -> tablet 进入 loading
     4. 非meta表，目标节点处于busy状态，打印日志，下一次循环调取去加载
  6. 截止到这里 tablet 处于 loading 状态
  7. 如果重试次数超过闸值                                                                           -> tablet 进入 LoadFail
  8. 如果 des down：
     1. 如果是meta表，根据size 选节点，然后绑定                    -> tablet 进入 offline
     2. 非meta表，如果des 还在内存中，绑定tablet 到 node   -> tablet 进入 offline
     3. 非meta表，如果des 不在内存，且为pendingoff状态     -> tablet 进入 offline
     4. 以上都不是 ，根据size 选节点，然后绑定                         -> tablet 进入 offline
     5. 会从第一部重新开始
  9. 如果已经发送load 请求，直接返回，等待下一次的调度，tablet状态不变
  10. load 成功                                                                                               -> tablet 进入 ready
  11. 如果是 ready 或者 loadfail 则流程结束，如果 tablet 不是ready，且tablet 重试次数小于闸值，则发起一个move流程

* TabletNode

  1. 在内存中（TabletManager）添加 Tablet，并创建 TabletIO。

  2. 调用 TabletIO 的 Load 接口，内部主要是 ：leveldb option 初始化（levledb配置）、调用leveldb 的 OpenDB 接口完成，LevelDB 的初始化。

     1. 根据 Schema 一个 LG 对应一个 LevelDB 实例，既LevelDB 的目录。
     2. LevelDB 的环境可以指定，可以是 DFS、Posix、MemStore等。
     3. 具体参数设置见[LevelDB 配置](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/options.md)，其中特别说明一下 parent_tablets 参数，如果该参数只有一个成员说明这是一个 Spilt 的请求，如果是两个成员变量说明是 Merge 参数。
     4. 由于 Tera 在 DFS 上面的 LevelDB 的[路径设置]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#sst-%E6%96%87%E4%BB%B6%E5%8F%8A%E8%B7%AF%E5%BE%84%E7%BB%84%E6%88%90](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#sst-文件及路径组成))，可以保证不同目录下的 SST 文件被正确索引，所以 Spilt 和 Merge 之后的 Tablet 文件可以来自不同的目录，然后在 Compact 阶段删除旧的 SST 文件，让新生成的 SST 文件在新的（可能是 Spilt、Merge）Tablet 目录下。

  3. Load 中的 LevelDB 配置见 [LevelDB Option](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/options.md) 和 [TabletNode 加载Tablet 源码解析](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/procedure/tn_load_tablet.md)。

     

##### 注意

1. 整体思路：Tablet 状态从 Offline -> Loading -> Ready，如果不是 Offline 其实状态，尽量转换为 Offlien状态（根据状态转换图），如果转换不了就异常结束流程。
2. 上述流程中在异常情况下，都会先判断是否是是 MetaTable（通过表名），如果是的话，**会优先加载 MetaTable**，保证 MetaTable 是正常的，否则更新都会失败。
3. **更新 MetaTable 的 Tablet 的元数据时，认为一定会成功的**，因为默认重试次数数无限次，即使 MetaTable 异常也会有限被加载，一旦加载成功则，更新 Tablet 的元数据一定会成功，所以默认是一定会成功的。
4. **更新 MetaTable 中 Tablet 的状态一定是 Offline 状态**，不会是其他的状态，其他的状态都是 Master 内存维护的状态，一旦 Master 切换了，会收集 TabletNode 中 Tablet 状态，在 Offline 状态的基础上做一些处理逻辑（前面介绍过），这是一种简化的时机模式（用集群恢复效率 避免 状态机维护的复杂性）。
5. Load 在最后执行失败了，会执行一步move的流程，因为此时可能load 成功，但是rpc返回失败，重新move的时候回 unload 和 load，保证只有一个节点load 最终成功。**保证了 Tablet 被唯一加载**。

5. create table 成功返回，load tablets 是异步，如果有load 失败，在哪里补偿回来呢？

   Load 失败会进行 Move，保证一定会成功。

6. 如何避免，多个TN 同时加载 Tablet？

   Load 失败会进行 Move，保证 Tablet 被唯一加载。



##### 源码解析

[Master Load Tablet 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/tablet/load_tablet_procedure.md)

[TabletNode 加载Tablet 源码解析](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/procedure/tn_load_tablet.md)



#### unload tablet

* Master
  1. 初始化一定是ready（ <- kTabletUnloading、kTabletUnloadFail），如果是 kTabletDelayOffline，转 Offline，Unload 直接成功。
  2. 如果 节点 down，转 Offline，Unload 成功  -> (注意心跳，delete node 部分，应该有tablet move逻辑)
  3. 截止到目前，Tablet 状态为 Ready。
  4. 节点正常：
     1. 是否处于 UnloadBusy（并行卸载不超过 50个 Tablet），空转等下次调度。
     2. 不处于 UnloadBusy，发送 unload 。                                      Ready -> Unloading
     3. 节点故障 或者 成功 或者 range不匹配 都算unload 成功。
     4. TN 处于busy ，则重试次数++， 找过闸值则 kick off 节点。
     5. 否则，无限制重试。
  5. 到此 处于 Unloading 状态。
     1. 未结束，空转等下一次调度。
     2. 少于重试闸值，Unload 成功。
     3. 如果处于 Kick TN 状态，空转等kick结果。
     4. 否则，UnLoadfailed。
* TableNode
  1. 在 TabletManager 中获取 TabletIO 内存数据。
  2. TabletIO Unload 逻辑：
     1. try_unload_count_++，用于判断是否处于 UrgentUnload 状态（闸值时 3次）。
     2. 调用 LevelDB 的 Shutdown1 接口，内部遍历所有的 DPImpl（LG）进行，对内存中的数据进行 Flush，切换 MemTable 落盘到 Level 0，并处理 wal。
     3. 然后等待 LevelDB 的引用计数为 1，因为 Shutdown1 期间还可能有数据写入，循环等待（周期100ms）。
     4. 停止 TabletIO 异步的写入（TabletNode 是批量写入 LevelDB，后面介绍），期间会将所有的写 Flush 到 LevelDB。
     5. 调用 LevelDB 的 Shutdown2 接口，Flush Shutdown1 阶段的写入数据。
     6. TabletIO 一些成员遍历的析构等处理。
  3. 删除内存（TabletManager）中的 Tablet 数据。

主要两阶段关闭

##### 注意

1. Master 在 Unload 流程中值更新了内存，没有更新 MetaTable。

   unload 在 move、mergr、spilt中不需要更新因为 Load 更新了。

   unload 在 Maser 主流程 都是清理无用tablet 不需要更新。

   unload 在 disable 中 已经disable 了 table，也不需要。

2. TabletNode 的 Unlaod 过程分位两阶段 Shutdown，因为写入时异步执行，既写入 和 Unload 可以并行，第二阶段保证 Unload 期间的写入会落盘。

3. TabletNode 并没有调用 DestroyDB 接口删除相应的目录，因为 Unload 之后一般会换一个节点重新 Load ，通常没有单独删除一个 Table 中某些 Tablet 的情况（因为 DFS 上的数据是一个整体在 Tera层来看）， 一般是 DeleteTable 阶段会删除相应的目录，这种情况系统也没有删除相应的目录或者文件（应该是一种保守做法，需要外围支持），只有在 DeleteTable 之后，重新创建同名的表会移除相应的目录到trash 目录的情况。



##### 源码解析

[Master Unload Tablet 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/tablet/unload_tablet_procedure.md)

[TabletNode Unload tablet](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/procedure/tn_unload_tablet.md)



#### move tablet

* Master
  1. 初始状态：
     1. 如果是 ready 或者 unloadfail 则 unload 再 load
     2. 如果是 offline 或者 loadfail 则 直接 load
  2. UnLoadTablet：Unload 源节点，此时结束时，Tablet 必须是 Offline 状态。有同步机制保证 Unload 成功再进行下一步。
  3. LoadTablet：走 Load 流程（如上所述）加载 Tablet。
  4. Eof：结束 RPC 请求。

##### 注意

1. 没有更新 MetaTable，在 Load 流程中处理， Unload 也不会删除原有的元数据，因为 MetaTable 是一个 KV 类型的表，在 Load 阶段会更新。
2. 如果 Master 切换，Unload 成功 Load 失败，MetaTable 保存的是 Unload 之前的 元信息，此时在 收集信息部分会处理（之前有介绍）。
3. Unload 有同步机制保证成功， Load 没有，说明 Move 成功了 Tablet 未必可用（未加载完毕），同时也保障了不会有两个节点同时加载 Tablet。



##### 源码解析

[Master Move Tablet 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/tablet/move_tablet_procedure.md)



#### spilt tablet

* Master

  1. 初始状态：一定是 Ready，否则失败。
  2. PreSplitTablet：spilt_key 范围校验，如果为空，会向 TabletNode 发送 ComputeSplitKey 请求，返回的结果，赋值给 spilt_key。
  3. UnLoadTablet：执行 Unload 流程，有同步机制保证，Unload 结束才进行下面步骤，
  4. PostUnLoadTablet
     1. 校验 Unload 是否完全结束，标准是查看 Tablet 的 wal 是否都消耗完毕（Split 必须保证 所有的日志都消费完）。
     2. 如果没有消费完，执行步骤7（FaultRecover），否则执行步骤5（UpdateMeta）。
  5. UpdateMeta
     1. 异步更新两个 Tablet （拆分后的两个 Tablet，主要是更改了 Key 范围）的元数据到 MetaTable（无限次重试）。
     2. 更新成功 MetaTable 之后，更新内存数据。
  6. LoadTablets：创建两个 Load 流程，加载两个子 Tablet（加载的节点相同，是拆分之前的节点）。
  7. FaultRecover：重新加载 父 Tablet，然后结束流程。
  8. Eof：结束 RPC 请求。

* TableNode

  TableNode 侧，在 Spilt Tablet 流程中主要接收三个请求，Load、UnLoad 之前都介绍过了，线面重点看一下 ComputeSplitKey ，ComputeSplitKey 的核心思想找到中间的 Key，中间的 Key 使得拆分之后 Tablet 的容量尽量均分。

  1. 在DBtable（Tablet）中对DBIpml（LG）根据大小进行排序，选择最大的DBIpml。
  2. DBIpml 调用 LevelDB 的  Version（SST 等元数据的内存维护类）类 计算（根据传入的大小占比，此处是0.5，既大容量50%的那个文件的最大key）。
  3. 从level 1（注意level 0 不算）开始，第一个文件的最大key开始往下逐层找到所有小于这个key的文件。
  4. 计算这些文件大小的和，如果小于目标容量（总容量 * 0.5），则对level 1的第二个文件的最大key最为比对，逐层往下找到所有小于他的文件进行累加。
  5. 3-4 逐层逐个文件文件计算，直到容量超过目标容量（总容量 * 0.5），返回split_key。
  6. 针对异常情况不是用容量划分，直接从key范围，根据字典序找到中间key。

  

##### 注意

1. 拆分前后的 Tablet 都在同一个节点。

2. 有一些重复，因为 Load 阶段还会更新 MetaTable。

3. Spilt 与之前的流程不同的是，先更新 MetaTable（一定成功，因为无限次重试），然后更新内存。

4. TableNode 的 ComputeSplitKey 从 Level 1 第一个文件的最大 Key 开始逐层划分的依据是， 从 Level 1 开始，文件间有序，这样划分之后的结果是，逐层划分为2，并且划分之后的两部分文件没有交集（利用了 LevelDB 中 SST 文件不变性 以及 同层 SST文件间有序且无交集）。

5. TableNode 选择容量最大的 LG 进行拆分，因为没有办法保证找到一个 Key 然所有的 LG（LevelDB实例） 都能正好分开，全局算法交复杂，涉及到跨 LevelDB 交互（跨线程）。

   这种办法可能导致拆分的结果不均匀，比如有多个 LG，最大的一个 LG 均分了，其他的 LG 根据这个 Key 拆分的结果 与 最大的拆分结果 结合，可能不是均分的。



##### 源码解析

[Master Spilt Tablet 流程](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/procedure/tablet/spilt_tablet_procedure.md)

[TabletNodeServer Compute Split Key](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/procedure/tn_compute_spilt_key.md)



#### merge tablet

* Master
  1. UnLoadTablets
  2. PostUnLoadTablets
  3. UpdateMeta
  4. LoadMergedTablet
  5. FaultRecover
  6. Eof



##### 注意



##### 源码解析





### Client Procedure

#### update(write/delete)

schema 的校验？

##### 注意

#### read

##### 注意

#### scan

##### 注意

#### transaction

##### single row transaction

##### 注意

##### global transaction

##### 注意

