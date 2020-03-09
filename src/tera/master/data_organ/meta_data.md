# Master 元数据组织

本部分看一下系统中的元数据组织形式，方便对系统的理解。本部分主要介绍以下两部分的元数据内容（Ts元数据Ts部分介绍）：

1. ZK中的元数据。
2. Master内存中的元数据。

系统的与数据和元数据都存储在下层的分布式文件系统中，所以Master的数据没有本地持久化。



### ZK中的数据组织

##### zk的数据内容

1. /tera/master-lock：master抢锁的节点。

2. /tera/master

   1. 抢锁成功之后，为主master，会将自己的地址写进当前节点。
   2. 注意：该节点为临时节点。

3. /tera/root_table

   1. 存储meta_table的地址。
   2. 系统中只有一个meta_table，记录所有table和tablet的元数据。

4. /tera/safemode

   1. 记录集群是否进入安全模式的标志位。
   2. 超过10%的tablet不可用时，集群进入安全模式（所有的表汇总之后的tablet，既安全模式针对一个集群，而不是针对一个表）。

5. /tera/ts

   1. Ts启动时会在该目录下注册节点。
   2. 子目录格式：uuid，uuid后十位是Ts的版本号，每次更新的时候回比较版本号。
   3. 子目录存储的数据内容：Ts的addr。

6. /tera/kick

   1. kick的节点会记录在该目录下。
   2. 子节点格式和数据内容与ts的子节点相同。
   3. 被kick的节点，由TabletNode节点负责删除（tabletnode部分介绍）。

   

##### watch zk中的数据变化

1. 删除节点（未列出的路径，表示忽略变化）
   1. /tera/root_table：从内存中获取meta_table的数据更新zk。
   2. /tera/master：释放锁，然后断开zk链接，程序会退出。
   3. /tera/ts：系统进入safe_mode模式。
   4. /tera/master-lock：程序退出。
2. 创建节点、节点数据变化：忽略。
3. 子目录变更
   1. /tera/ts：更新内存中Ts等数据。



### Master内存中的元数据

Master内存中的元数据，分两个线条进行管理。一个是TabletNode，一个是Tablet，tablet中记录addr把两个线条的元数据进行关联。下面分这两部分进行介绍。

#### TableNodes

1. TabletNodeManager内部维护tabletnode_list_成员变量。
2. tabletnode_list_：类型std::map<std::string, TabletNodePtr>，addr->node 的map映射。
3. TabletNode包含的数据：
   1. std::string addr_：节点的地址
   2. std::string uuid_：节点的uuid标识，互10位是节点的版本号。
   3.  NodeState state_：节点的状态，包括：kReady、kOffline、kPendingOffline、kOnKick、kWaitKick、kKicked（状态机的装换在后面介绍）。
   4. int64_t timestamp_：节点的创建时间，节点必须创建60s之后才能进入kOffline，否则kPendingOffline。
   5. StatusCode report_status_：心跳上报的节点状态，包括：kTabletNodeNotInited、kTabletNodeIsBusy、kTabletNodeIsIniting、kTabletNodeIsReadonly、kTabletNodeIsRunning。
   6. TabletNodeInfo info_：pb结构，心跳上报的数据，用来更新当前结构的成员变量。
   7.  uint64_t data_size_：节点上所有tablet数据的总和，可不是磁盘空间。
   8. uint64_t qps_：所有tabletqps的总和。
   9. uint64_t load_：节点上报的负载。
   10. uint64_t persistent_cache_size_：节点上报的cache大小，在TS上的存储，其余的应该都在BFS上。
   11. uint64_t update_time_：最后一次更新数据的时间。
   12. std::map<std::string, uint64_t> table_size_：tablename->table-size的map映射，既表维度下的统计。
   13. std::map<std::string, uint64_t> table_qps_：tablename->table-qps的map映射，既表维度下的统计。
   14. MutableCounter average_counter_：read_pending_、write_pending_、scan_pending_、row_read_delay_统计数据的计算结果，前一个周期站三分之一的权重，当前周期占三分之二的权重。
   15. uint32_t query_fail_count_：心跳探测的失败次数，超过10次进行kick，有一次成功则清零。
   16. uint32_t onload_count_：记录当前节点正在load的tablet数量，并发数不能超过20个。
   17. uint32_t unloading_count_：记录当前节点正在unload的tablet数量，并发数不能超过50个。
   18. uint32_t onsplit_count_：记录当前节点正在split的tablet数量，并发数不能超过1个。
   19. uint32_t plan_move_in_count_：记录当前节点正在move in的tablet数量，并发数没有限制。
   20. std::list<int64_t> recent_load_time_list_：记录最近load tablt的时间点20个，最旧的时间点超过300s才能继续load，加载几点在并发度和频率上都有所控制。



#### Tablet

1. TabletManager内部维护all_tables_、meta_tablet 成员变量。

2. meta_tablet：类型MetaTabletPtr，是元数据表的信息，与普通的tablet相同只是表名规定为meta_table。

3. 注意：meta_tablet是单独管理的，不再某一个Table里，在split_tablet时，不再查找返回，所以meta_table是连续的永远不会拆分。

4. all_tables_：类型<std::string, TablePtr>，既表名->表的映射。

5. Table

   1. std::map<std::string, TabletPtr> tablets_list_：start_key-> Tablet的映射。

   2. std::string name_：表名。

   3. TableSchema schema_：表的schema，是pb结构。

   4. std::vector<uint64_t> snapshot_list_：未使用。

   5. std::vector<std::string> rollback_names_：未使用。

   6. uint32_t deleted_tablet_num_：删除的tablet数量，当与tablets_list_大小一致时，需要删除表。s

   7. uint64_t max_tablet_no_：最大的tablet号，单调递增，每次申请新的tablet时+1。

   8. const int64_t create_time_：表格创建时间。

   9. TableCounter counter_：table的统计信息，pb结构，没5s刷新一次。

      1. ```protobuf
         message TableCounter {
           optional int64 lread = 1;        // 所有tablet的low_read_cell之和
           optional int64 scan_rows = 2;    // 所有tablet的scan_rows之和
           optional int64 scan_max = 3;     // 所有tablet的scan_rows最大值
           optional int64 scan_size = 4;    // 所有tablet的scan_rows之和
           optional int64 read_rows = 5;    // 所有tablet的read_rows之和
           optional int64 read_max = 6;     // 所有tablet的read_rows最大值
           optional int64 read_size = 7;    // 所有tablet的read_size之和
           optional int64 write_rows = 8;   // 所有tablet的write_rows之和
           optional int64 write_max = 9;    // 所有tablet的write_rows最大值
           optional int64 write_size = 10;  // 所有tablet的write_size之和
         
           optional int64 tablet_num = 20;   // 所有的tablet数目之和
           optional int64 notready_num = 21; // not ready状态的tablet数目之和
           optional int64 size = 22;         // ready状态的tablet size之和
           repeated int64 lg_size = 23;      // ready状态的lg size之和
         }
         ```

   10. TableMetric metric_：table统计信息（tablet维度），没5s刷新一次

       1. table_name_：表名。
       2. tablet_num_：tablet数量。
       3. not_ready_：非ready的tablet数量。
       4. table_size_：table的大小。
       5. corrupt_num_：损坏的tablet数量。

   11. bool schema_is_syncing_：schema切换的时候使用。

   12. TableSchema* old_schema_：schema切换时，使用双buffer，回滚时使用。

   13. std::map<uint64_t, std::map<TabletFile, InheritedFileInfo> > useful_inh_files_：tablet_id -> <sst文件 -> 文件的引用计数结构体>的双层映射，需要被gc的文件会加入其中。

       1. InheritedFileInfo：Master内存中对dfs中文件（sst文件）的引用计数，既Master要对一下文件进行使用进行虚拟的一层引用计数，不是底层dfs真实的引用计数。
          1. ref：引用计数。
       2. TabletFile：sst文件
          1. tablet_id：tablet id。
          2. lg_id：lg id，在ts部分介绍。
          3. file_id：文件id，应该是sst文件前面的序号。

   14. std::queue<TabletFile> obsolete_inh_files_：需要真实删除底层dfs的文件会放入其中，当useful_inh_files_中一个table_id内所有的sst文件引用计数都为0时，该table的文件会移入obsolete_inh_files中，并且从useful_inh_files_中移除。

   15. std::set<uint64_t> gc_disabled_dead_tablets_：记录需要gc的talet_id。

   16. uint32_t reported_live_tablets_num_：心跳检测上报的活跃的tablet数量，内存中在merge、spilt、gc时都会修改该数据。

   17. TableStateMachine state_machine_：table的状态机，后续介绍状态转换。

   18. bool in_transition_ = false：表的锁机制的标志，表的一些事物操作需要。

6. Tablet

   1. TabletMeta meta_ ：pb结构上报的tablet的meta信息，详见[tabelt](./tablet.md)。
   2. TabletStateMachine state_machine_：节点状态机，详见[tabelt](./tablet.md)。
   3. TabletNodePtr node_：所在的Ts节点。
   4. TablePtr table_：所在的表。
   5. int64_t update_time_：心跳返回包，更新tablet数据的最新事件。
   6. int64_t last_move_time_us_：最新的move tablet 时间。
   7. int64_t data_size_on_flash_：在flash的数据大小，猜测在TS上的数据大小。
   8. std::string server_id_：未使用。
   9. std::vector<std::string> ignore_err_lgs_：lg的忽略错误（在ts部分介绍）。
   10. std::list<TabletCounter> counter_list_：未使用，统计中使用的average_counter_，既ts的上报数据。
   11. TabletCounter average_counter_：前一个周期站三分之一的权重，当前周期占三分之二的权重，与ts的策略相同。
   12. struct TabletSplitHistory split_history_：tablet spilt的时间间隔必须是600s。
   13. bool in_transition_ = false：tablet事务使用。
   14. bool gc_reported_：gc上报的标志。
   15. std::multiset<TabletFile> inh_files_：tablet使用的文件（dfs）。
   16. std::atomic<int> load_fail_cnt_：未使用。
   17. const int64_t create_time_：创建时间。

7. Tablet的很多元数据在TabletMeta中，包括节点id、路径等，详见上述链接。

8. 横向限制策略的梳理：

   1. load：节点 < 20，记录20个时间段，最久的超过5min；
   2. unload：节点 < 50。
   3. spilt：节点 <= 1，同一个tablet 必须间隔 10min以上。
   4. merge：节点无限制
   5. move in：节点无限制



### 总结

本部分的内容相对较多，第一次看的时候时候可能很难抽象理解，但是看过一遍相信对后面的流程逻辑的理解会方便一些，到时候再回头看就会加深对系统的理解。