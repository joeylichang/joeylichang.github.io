* **Navigation**
  * [TabletNode Arch](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/tablenode_overview.md#tabletnode-arch)
    * [TableNode Class Arch]()
    * [TableNode Thread Arch]()
  * [Leveldb Opt](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/tablenode_overview.md#leveldb-opt)
  
  
###  TabletNode Arch

##### TableNode Class Arch

<img src="../../../images/tera_tn_arch.png" alt="tera_tn_arch" style="zoom:50%;" />

TabletNode 主要的类关系图如上所示，TabletNodeEntry 是 TabletNode 的入口，MetricHttpServer 提供 http 接口用于外部监控系统获取其统计信息，RemoteTabletNode 是 RPC 回调的入口后续介绍。

TabletNodeImpl 是 TableNode 的核心，内部主要的成员变量是 TabletManager 负责管理当前节点全部的 Tablet，TabletIO 可以理解为一个 Tablet 的对象，其主要负责与 LevelDB 的交互，涉及到读、写、Scan的核心逻辑。其他的类都是一些辅助，后续涉及到会详细介绍。



##### TableNode Thread Arch

默认配置 TabletNode 的线程分配情况如下：

1. 写请求线程池，10个线程。
2. 读请求线程池，40个线程。
3. Scan请求线程池，30个线程。
4. Compact请求线程池，30个线程。
   1. 每个 LevelDB ，既每个 lg 有8个 compact 线程。
5. lightweight_ctrl_thread_pool_，默认 10个线程。
   1. Query、Update、CmdCtrl、ComputeSplitKey、loadtablet、unloadtablet 部分情况（已经在执行，可能客户端那边有重试，ctrl_thread_pool_ 任务没有及时处理时）使用当前线程池。
6. ctrl_thread_pool_，默认20个线程。
   1. loadtablet、unloadtablet 优先使用该线程池。

加上主线程，默认情况 TabletNode 需要至少 141 + tablet_num * lg_num * 8 个线程。



##### 源码解析

* [TabletNode 类架构](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/init/tn_arch.md)
* [TabletNode RPC](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/rpc/tn_rpc.md#remotetabletnode)



### Leveldb Opt

##### LevelDB Arch

<img src="../../../images/tera_leveldb_class.png" alt="tera_leveldb_class" style="zoom:33%;" />

Tera 对LevelDB 进行了一些改动目的是支持系统的设计需求，上面是改动后的 LevelDB 层主要的类关系图。

主要改动如下：

1. 增加 DBTable 类继承DB，DBTable是对tablet的抽象，DBTable内部封装了DBImpl，DBImpl对lg进行抽象。
2. OpenDB 进行了改动，支持 Spilt Tablet、Merge Tablet（如果 option 有 parent_tablet 参数，说明是 Spilt（1个parent_tablet） 或者 Merge（2个parent_tablet））。
3. DBImpl 的压缩进行改动，引入压缩策略，在压缩过程删除各种类型的 Key（例如：TKT_DEL_QUALIFIER、TKT_DEL_QUALIFIERS、TKT_DEL_COLUMN、TKT_DEL、版本号等）。
4. 支持多线程Compact（默认8个线程）。
5. 对ENV进行继承，支持多套环境，包括HDFS、AFS（百度自研）、Local等（本系列关注HDFS）。
6. 增加PersistentCache，利用本地SSD磁盘缓存 SST 文件。



##### LevelDB Cache

<img src="../../../images/tera_leveldb_cache.png" alt="tera_leveldb_cache" style="zoom:50%;" />

Tera LevelDB 的缓存架构如上图所示，主要分两层三块。两层分别是 内存 和 SSD 磁盘。三块分别是 TableCache 、BlockCache、PersistentCache。其中 PersistentCache 在 SSD 磁盘上。

TableCache 存储的是 key->db_name+file_name，value->sst的索引数据。读取时从里面读，不再的时候会从dfs读出来写入。level0 落盘时，会写入。BlockCache 存储的是用户数据。读取时从里面读数据，不再的话会从DFS中读数据写入。level0 落盘时，会写入。PersistentCache 存储的是 SST 文件。

正如前面介绍过，缓存这部分利用了 SST 文件的不变性极大了减少了这部分设计的复杂度（只有淘汰，不用考虑失效的问题，避免了 Write Back 或者 Write Through 的复杂性）

详情见[Tera-LevelDB缓存]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/leveldb.md#tera-leveldb%E7%BC%93%E5%AD%98](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/leveldb.md#tera-leveldb缓存))、[PersistentCache](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/persistent_cache.md)。



##### Read

1. 在mutable和immutable中查找。

2. 在TableCache中查找(TableCache缓存的内容：key->db_name+file_name，value->sst的索引数据)。

   1. 如果在TableCache中，直接去BlockCache（缓存的用户数据）中get数据。

      1. 如果不在 BlockCache， 根据TableCache中的索引数据去sst文件中读数据。
      2. 读回的数据再重新插入BlockCache。
      3. 如果系统支持PersistentCache，并且该请求有回填选项，并且内容不是从PersistentCach获取，则调度将dfs文件读到本地。

   2. 如果不在TableCache中

      1. 打开dfs中相应的sst文件。
      2. 如果在PersistentCache中，从PersistentCache中读取index数据。
      3. 如果不在PersistentCache，或者没有使用PersistentCache，从dfs中读取sst文件的index数据。
      4. 根据读取到的索引数据，在内存中组织成Table数据结构，然后写入TableCache。

      

##### Write

1. 先判断 WriteOptions 是否设置了 disable_wal（可以设置不写 log，直接写数据），如果没有设置 disable_wal，判断 log 是否超出了设置的大小（默认 32M），如果超出则，switch log。
2. 如果没有设置 disable_wal，则先写 log
   1. Log 可以通过 WriteOptions 设置 sync 标志是否同步到磁盘（DFS也是支持的）。
   2. **注意**：目前都是在 DBTable 层的操作，既属于一个 Tablet 的所有 lg 共用一个 Log。
3. 根据写请求中的 RawKey 对其进行拆分，拆分到不同的 lg，既 DBImpl（LevelDB 实例）。
4. 遍历上述拆分的请求，分别对相应的 lg，进行写操作：
   1. 第一次遍历：调用 DBImpl 的 GetSnapshot，对所有需要更新的 lg，设置一份快照（目的是不影响写期间的读）。
   2. 第二次遍历：
      1. 先检查 memtable 是否有足够空间，没有进行切换，如果 immemtable 也是满的就等 min compact。
      2. 写入 memtable。
   3. 第三次遍历：删除之前 GetSnapshot 返回的快照。
5. 通过信号量机制，串行的进行下一个写操作。



##### Compact

除了 LevelDB 自身根据打分进行的 Compact，Tera 还支持客户端通过接口调用执行 Compact（启动新的线程，分 lg 并行进行）。



##### LevelDB 解析 && Option

LevelDB 更新详细的介绍见：

* [TabletNode中的LevelDB](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/leveldb.md)。
* [PersistentCache](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/persistent_cache.md)

Tera 对 LevelDB 有了较大的改动，也增加了一些配置，在对LevelDB的有了了解之后，有必要了解一下其配置，对于性能调优、线上问题定位、监控理解等方面都是有必要的。

* [LevelDB 配置](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/options.md)

更多关于 LevelDB 的介绍见[LevelDB解析](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/leveldb/overview.md)。



### TabletNode RPC Arch





### Load Tablet





### UnLoad Tablet

主要两阶段关闭



### Statistic



### Query（HeartBeat Response）



### TabletNode Period Task



