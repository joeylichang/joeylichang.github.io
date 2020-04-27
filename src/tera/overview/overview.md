* **Navigation**
  * [System Arch](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#system-arch)
  * [Leveldb Character](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#leveldb-character)
  * [Consistency](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#consistency)
  * [Data schema](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#data-schema)
  * [Meta Data](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#meta-data)
    * [Zk Meta](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#zk-meta)
    * [MetaTable(t)](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#meta-tablet)
    * [Dfs Dir](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md#dfs-dir)



### System Arch

Tera是百度开源的Table存储系统，主要用于网页和链接的存储，在链接选取、爬虫抓取、单网页计算、多网页计算、索引建库等整个搜索流程中起到非常重要的作用，在实时性 和 性能上与前一代MR + HDFS的架构有了较大提升。系统架构图如下所示：

<img src="../../../images/tera_arch.png" alt="../../../images/tera_arch.png" style="zoom:30%;" />

Tera 的设计思路大部分借鉴了Google BigTable的设计思想，系统主要分为两层，底层数据的持久化基于BFS（百度内部的文件系统，内部已经用新一代的自研AFS替换了BFS）同时兼容HDFS，上层是基于分布式文件系统（例如HDFS）实现的Table存储。

TabletServer是用户数据的存储节点，内部使用了LevelDB，并进行了大量优化（或者说改动，目的是支持Tera的设计需求），将log（wal） 和 sst（用户数据）都持久化到HDFS上（leveldb优化的详情见TableServer介绍）。

Master是系统的协调节点负责table的create、disable、enable、delete、update，tablet的load、unload、spilt、merge、move，集群负载均衡，TabletNode管理，GC等。Master维护的元数据不是在内存中同样存储在TabletNode中（既底层的分布式文件系统），系统中默认会创建一个MetaTable用于存储系统中所有table的元数据（后面元数据部分有介绍）。

架构图中显示Master有多个备份，Master通过ZK选主，从Master内存并没有元数据信息，而是在主Master丢失了ZK的Lock之后，新的主Master从MetaTable加载元数据到内存。以上设计的根据正是元数据也持久化在分布式文件系统中，元数据的恢复是秒级操作（对系统可用性影响不大），同时降低了Master复杂度（避免了Raft、Paxos等一致性算法的引进）。

Client提供了事务（单行事务、全局事务）的能力，同时需要与Master交互完成元数据的获取，再进一步完成用户数据的路由，在全局事务中Client还会和ZK进行交互。

Tera 是完全由C++实现，相比HBase在性能、GC上更有优势（实时处理），社区目前基于C++的类BigTable的项目 Tera是唯一一个经过大数据量、大流量工业环境考验的系统。同时Tera 还支持BigTable 中 Locality Group 概念，相比HBase在多业务场景更有优势（例如，多业务修改表的不同列簇时，同一个业务关心的列簇被集中处理，后面会详细介绍LG）。



### Leveldb Character

TabletNode 内部使用的leveldb作为单机的存储引擎（详情见[leveldb](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/leveldb/overview.md)），对其ENV进行了分布式文件系统的扩展。TabletNode 充分了利用了 leveldb 中 sst（新版本是ldb文件） 文件不变性这一特点，既 leveldb 中的 sst 文件一旦生成不会有改动的情况发生，leveldb内部通过compact对多个sst进行多路归并生成新的sst文件，然后删除旧的sst文件，没有更改sst文件内部数据的情况发生。

Tera 正式利用了leveldb 内部 sst文件的不变性极大的减少了复杂性 和 优化了性能。TableNode 上有两层缓存，一层是内存的TableCache 和 BlockCache（leveldb原生，也有改动见后面TableNode部分介绍），另一程是基于SSD盘的PersistentCache，PersistentCache在本地缓存的sst文件永远有效（不存在失效的情况），由于sst的不变性，只要leveldb内存中的元数据（或者叫索引信息）指向了某一个sst文件且在PersistentCache中，直接读取即可，不存在PersistentCache 与 DFS（分布式文件系统）数据不一致的情况，避免了传统缓存的write back、write through带来的复杂度（BlockCache 同样）。

LevelDB 中的sst文件除了不变性，其内部的数据也是有序的，Tera利用这一点加速了Tablet（Table水平切分的单元，也是TableNode的数据组织单元）的分裂与合并。LevelDB 内存的信息用Version 和 VersionSet进行组织（详情见之前的[LevelDB介绍的文章](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/leveldb/data_type.md)），Tablet加载时的内存数据通过MANFEST文件（存储在DFS）获取，TableNode维护内存数据，LevelDB的文件（SST、MANFEST、LOG、CURRENT等）都在DFS，所以Tablet加载和卸载的过程是从DFS读取元信息加载到TableNode内存（以上见之前[LevelDB](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/leveldb/overview.md)介绍文章即可）。Tablet分裂时，计算出分裂位置（分裂的Key），在TableNode加载相关SST文件即可。合并的Tablets其key必须是连续的，同样加载两个Tablet的sst文件到共同的TableNode内存即可。顾Tablet的分裂 与 合并都没有文件或者数据的copy 和 move都是很轻量级的实现。

Tablet的加载、卸载、分裂、合并在后续还会重点介绍，在这里需要了解的是Tera 利用LevelDB中sst文件的不变性和有序性极大的优化了性能。这里有一个问题需要思考，两个Tablet对应leveldb两个目录，如何保证能在一个TabletNode的内存中一起加载，Tera在这里做了两部分工作，第一对Tablet目录的命名进行了设计，保证不同的目录的sst文件可以被正确的加载和获取到，第二是GC的设计保证了不会误删 和 漏删 用户的数据（后面都会详细的介绍）。



### Consistency

##### 用户态读写一致性

Tera的数据多备份由底层的DFS进行保证（一般是3备份），Tera层的TabletNode是单副本。TabletNode可以理解为 DFS 层数据的索引数据。索引数据的加载是秒级别，并且Tablet支持快速的分裂和合并，所以单副本是完全支持业务的需求（BigTable 的设计同样如此）。所以一旦写成功，就可以读到最新的数据（可以理解为强一致模型）。

Tera 会将用户的数据以行为单位映射为KV进行存储（下面的Data Schema会详细介绍），这就会引入一个问题，既一行数据只更新了某几列，会不会读到一行数据其部分列更新了部分未更新或者更新失败的问题，Tera 内部实现上保证行级别的原子更新（后续文章有介绍）。

TabletNode 内是读写分离的，如何保证读不到残行呢（上述情况）？答案是快照（LevelDB支持快照），在每次更新时，会创建一个快照，写成功之后删除快照，写期间的读都是基于此快照。



##### Master 内存 与 MetaTable 一致性

元数据会在Master内存 和 MetaTable 都进行存储，引进来的问题是如何保证其一致性，操作都会先更新MetaTable再更新内存，更新MetaTable可以认为一定会成功，因为MetaTable异常会有限被迁移到其他TabletNode进行加载，Master可以异步更新MetaTable 并无限次重试。

存在一种情况，Master 异步更新MetaTable，然后更新了内存，还未等异步更新MetaTable成功既故障了，发生了主从切换，从MetaTable读取的meta不是最新的，这种情况会通过Master 的 心跳 以及 元数据本身进行修复（详细情况加Tablet Load的介绍）。



##### TabletNode 与 DFS 一致性

TabletNode 与 DFS的一致性，在进行更新操作时主要是两部分，第一是内存更新用户数据（可以认为不会失败），第二是log更新到DFS上（先更新log，后更新内存），log更新成功此次更新请求认为更新成功，会严格保证一致性。TabletNode 的 Cache 如上所述，利用了sst文件的不变性不存在不一致的情况。



### Data Schema

Tera 的数据模型完全参考 BigTable，其数据组织情况如下图所示：

<img src="../../../images/tera_data_schema.png" alt="tera_data_schema.png" style="zoom:50%;" />

与传统的表模式相比，Tera支持以时间戳为版本号的多版本存储，既上述三维空间。Tablet 是可以理解为一个 Table 的水平切分，既一行数据一定在一个Tablet内。一个 TabletNode 对应多个 Tablets（可以是来自不同的Table）。一张表格可以理解为如下的多级map： 

```C++
map<RowKey, map<ColummnFamily:Qualifier, map<Timestamp, Value> > > 
```

RowKey 代表 user_key 也是 Tablet 分区管理的依据。Qualifier 是 列的概念，ColummnFamily 由多个Qualifier组成的列簇，Timestamp 代表数据的时间戳由 Client 传进来，Value是用户存储的数据。LevelDB存储的RawKey = 编码（RowKey + ColummnFamily + Qualifier + Timestamp + Type） （编码规则后续介绍）。

Tera 除了上述逻辑划分的概念 还支持 Locality Group 物理划分，Locality Group 是由多个 ColummnFamily 组成（ColummnFamily 必须属于一个 Locality Group），一个 Locality Group 对应一个 LevelDB 实例（一个LevelDB目录）。Locality Group 可以加速 经常一起处理的 ColummnFamily 性能。一个 Table 可以划分若干个   Locality Group，同一个表格里的所有 Tablets 都会有相同数量的 Locality Group，既LevelDB，每一个LevelDB内负责不同区间的RowKey。

如果所有的 CF 属于一个 LevelDB 实例，访问一行中的某些列，会浪费资源很多无用的列，如果每个 CF 用一个 LG（LevelDB实例），如果访问一行需要每个 LevelDB 实例都进行遍历，效率也不高。LG 是一种折中方案，并且属于一个 LG 的列可以进行集中优化（比如：根据业务数据特点，相似数据的列放在一起可以提高压缩比例，再比如经常访问列放在一起可以放入内存优化等）。



### Meta Data

如上述架构所述，Tera的元数据主要包括Metatable、ZK。但除此之外 DFS上的目录也是需要关注的，因为他设计到 Tablet 的 spilt 和 merge。

#### Zk Meta

```C++
|_tera
      |_master-lock   // master 抢主的锁，抢到锁的master为主master，否则会一直重试不再进行内存元数据加载
      |_master        // 临时节点，抢到主的 master 会注册自身的信息（ip:port）到该节点下
      |_root_table    // 负责 MetaTable 的 TN 的信息（ip:port）
      |_safemode      // 是否处于 安全模式 的标志位
      |_ts            // TN 节点注册的目录，注册节点信息（ip：port）
      |_kick          // master kick_off 的节点会注册在该节点下（后面部分会详细介绍，节点的状态转化）
```

safemode：（master）判断是否进入safemode的标准，故障 TN 上的 Tablet 是否占总数的10%以上（默认配置）。

ts：TN 注册的临时节点的父目录，其子节点格式  {session_id}#{sequential_ephemeral_id}，session_id 是 TN 与 ZK链接的session_id，后面10位是全局连续递增的id，Master在汇总TN 信息时，会比较后10位的id，保证最新的TN被更新到内存（防止节点故障重连之后旧节点仍然存在的问题）。

kick：Master 在一些异常情况下 会认为节点故障，对其进行KickOff，会将TN下面相应的节点在kick目录下存一份待后续处理（Master部分会详细介绍）。TN 会一直 watch 自身在kick 目录下的节点，如果发现被master kick off，会删除该节点，告诉 master 自身是正常运行的，如果 TN 真的故障 kick 目录下会一直保存曾经故障的节点（方便后续问题定位）。



##### Master 关注的 zk 事件

1. /tera/root_table 被删除，会用内存中的数据更新zk。
2. /tera/master-lock 被删除，退出程序。
3. /tera/master 被删除，master会断开连接（释放master_lock），退出程序，让其他master重新抢主。
4. /tera/ts  被删除，标记集群进度safemode。
5. /tera/ts 有变更，会更新内存中的数据。

详情见[Master中zk元数据部分]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/meta_data.md#zk%E4%B8%AD%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%84%E7%BB%87](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/meta_data.md#zk中的数据组织)) 和 [Master-ZK 交互逻辑]([https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/zk_master.md#master-zk-%E4%BA%A4%E4%BA%92%E9%80%BB%E8%BE%91](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/logic/init/zk_master.md#master-zk-交互逻辑))



##### TabletNode 关注的 zk 事件

1. /tera/safemode 被删除，TN 退出只读状态。
2. /tera/safemode 被创建，TN 进入只读状态。
3. /tera/root_table 被删除，重新watch 等到被重新创建之后在更新内存，否则不改动内存。
4. /tera/root_table 被创建，设置root_table信息（ip:port）到内存。
5. /tera/root_table 变更，更新内存中root_table的信息。
6. /tera/ts/ 下自身注册的节点被删除，关闭 zk 链接，重连。
7. /tera/kick/ 下自身的节点被创建，删除相应节点，并退出程序。



#### MetaTable(t)

MetaTable 是系统默认创建的用于存储全局的元数据，MetaTable 不是上述的表格结构的表示KV结构（Tera也支持KV结构，并且性能明显优于表格式的结构），MetaTable 具有以下特性：

1. KV 结构，而非表结构。
2. 只有一个Tablet，既MetaTablet。
3. 不可以进行合并 、 分裂、因负载均衡而搬迁，并且在所有的流程中（Tablet 加载、搬迁等）都会优先被处理。
4. MetaTable 可以设置 与其他表在物理上进行隔离，既单独的 TN 只负责MetaTable。
5. MetaTable 在DFS上的目录与普通的Table 也是不一样的（后面介绍）。
6. MetaTablet  初始化 或者异常情况下，会根据Load Balance 中容量的策略（Master部分介绍）选取一个TN 进行加载。

下面看一下 MetaTable 都存储一些什么信息。 MetaTable 主要存储的数据包括： 所有的 Table 以及其 Tablet 的信息，除此之外还有用户、权限、Quta等信息，此处我们重点看一下 Table 以及其 Tablet 的信息。

###### Table 元数据

MetaTable 是 KV 结构，存储 Table 的元数据时：

Key = "@" + ${table_name}

Value = SerializeToString(TableMeta)

```protobuf
message TableMeta {
  optional string table_name = 1;		// table name
  optional TableStatus status = 2;	// table 状态 kTableEnable、kTableDisable、kTableDeleting 后续有介绍
  optional TableSchema schema = 3; 	// 表的schema
  optional uint64 create_time = 5;	// 创建时间
  optional AuthPolicyType auth_policy_type = 8; // 默认ugi 后面权限部分有介绍
}

message TableSchema {
    optional int32 id = 1;
    optional string name = 2;
    optional int32 owner = 3; // deprecated
    repeated int32 acl = 4;
    repeated ColumnFamilySchema column_families = 5;	// cf 的schema, 必须属于某一个lg
    repeated LocalityGroupSchema locality_groups = 6;	// lg 的schema 
    optional RawKey raw_key = 7 [default = Readable];	// leveldb 存储的key 类型 Readable、Binary、TTLKv、GeneralKv
    optional int64 split_size = 8; // MB
    optional int64 merge_size = 10; // MB
    optional bool disable_wal = 11 [default = false];	// 是否写leveldb 的log
    optional string admin_group = 12; // users in admin_group can admin this table
    optional string alias = 13; // table alias
    optional string admin = 14;
    optional bool enable_txn = 15 [default = false];	// 单行事务 和 全局事务的标志
    optional bool enable_hash = 16 [default = false];	// user_key 路由到tabletNode 是否使用hash
    optional uint32 bloom_filter_bits_per_key = 17 [default = 10];

    // deprecated, instead by raw_key GeneralKv
    optional bool kv_only = 9 [default = false];	// metatable 是 kv_only，用户表根据业务需要
}

message ColumnFamilySchema {
    optional int32 id = 1;		// cf 的 id
    optional string name = 2;
    optional string locality_group = 3;	// 所属的lg
    optional int32 owner = 4;
    repeated int64 acl = 5;
    optional int32 max_versions = 6 [default = 1];
    optional int32 min_versions = 7 [default = 1];
    optional int32 time_to_live = 8 [default = 0]; // 单位:秒(0:不过期, <0:提前过期, >0:延后过期)
    optional int64 disk_quota = 9;
    optional string type = 10;
    optional bool gtxn = 11 [default = false]; // 'gtxn=on' for global transaction feature availability 
    optional bool notify = 12 [default = false]; // 'notify=on' for notify feature availability
}

message LocalityGroupSchema {
    optional int32 id = 1;
    optional string name = 2;
    optional StoreMedium store_type = 3 [default = DiskStore];	// 存储介质，一个lg 对应一个leveldb实例，后面有介绍
    optional bool compress_type = 4 [default = false];
    optional int32 block_size = 5 [default = 4]; // KB
    optional bool use_bloom_filter = 6;
    optional bool is_del = 7 [default = false];
    optional bool use_memtable_on_leveldb = 8 [default = false];
    optional int32 memtable_ldb_write_buffer_size = 9 [default = 1000]; //KB
    optional int32 memtable_ldb_block_size = 10 [default = 4]; //KB
    optional int32 sst_size = 11 [default = 8388608]; // Bytes
}

```



##### Tablet 元数据

存储 Tablet 的元数据时：

Key = table_name + ‘#’ + key_start

Value = SerializeToString(TabletMeta)

TabletMeta 解析详见[TabletMeta 数据解析](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/master/data_organ/tablet.md#tabletmeta)



#### Dfs Dir

##### sst 文件及路径组成

```c++
../data/${tablename}/tablet${tablet_no}/${lg_no}/${tablet_no}(4B)${filenumber_no}(4B).sst
```

../data/ ，是默认的跟路径，可配置

../data/${tablename}，  表示同一个Table的Tablet 存储在一个目录下

../data/${tablename}/tablet${tablet_no}/， Tablet是Table的子目录

../data/${tablename}/tablet${tablet_no}/${lg_no}/， 一个lg 对应一个leveldb 实例，所以这个目录才是leveldb操作的目录，MANFEST、CURRENT、SST文件都存储在这个目录里。

注意：Tablet 的所有lg 公用一个log，所以log文件存储在../data/${tablename}/tablet${tablet_no}/ 目录里。

tablet_no(4B)filenumber_no(4B).sst，sst文件名的组成是tablet_no + filenumber_no 组成，filenumber_no是全局递增的，这种命名方式是很有考究的，因为根据文件名就能获取所属的Tablet，在通过tablet_no 可以拼装成整个文件的路径。

在 Tablet 的分裂过程中，只是将同一个 Tablet 的文件根据spiltkey（计算策略后面介绍）进行划分在 TabletNode 被两个拆分后的 Tablet 重新加载即可，根据其文件命名就可以准确的定位到文件的目录，在分裂后的Tablet进行compact时，删除旧的sst文件，在新创建的两个Tablet中生成新的sst文件。整个过程没有文件的copy 和 move 极大的提升了 分裂与合并的性能。配合系统的GC 模块（后续介绍）可以保证文件被正确删除，不会漏删或者误删。



##### MetaTable 的目录

```c++
../data/${tablenamt}/meta/0
```

MetaTable 是系统默认创建的，使用特殊的目录是方便管理 和 随时被 TabletNode 进行加载。并且 MetaTable 只有一个 lg（0）。



##### trash 目录

```c++
../data/#trash/tablename + "." + time
```

Table 在创建开始之前，会将之前同名的 Table 相关数据 move到该目录下，GC阶段会清除该目录内所有的数据。 



```c++
../data/#trackable_gc_trash/table_name/tablet + tablet_no/lg_id/file_id.sst.time
```

要删除的sst文件会按照上面的格式先保存在trackable_gc_trash 目录下保存7天（默认配置），Master GC阶段会定时check 如果过期则会删除。

