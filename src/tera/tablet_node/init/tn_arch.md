# TabletNode 类架构

### 类图

<img src="../../../../images/tera_tn_arch.png" alt="tera_tn_arch" style="zoom:50%;" />

##### TabletNodeEntry

程序入口，RpcServer负责rpc通信（定制化调度策略），RemoteTabletNode负责管理所有RPC的远程调用入口，TabletNodeImpl是TableNode的核心逻辑，MetricHttpServer负责TabletNode统计数据的获取（TabletNodeImpl会注册相应的统计项）。

##### TabletNodeImpl

TabletNodeImpl 主要的成员变量是TabletManager，TabletManager是对TabletIO做了一层简单的封装，负通过map管理多个TabletIO。TabletIO是Tablet的抽象直接负责与LevelDB的交互。

除了TabletIO，还有一些周期性的统计工作（例如TabletNodeSysInfo、CacheMetrics、AutoCollectorRegister），以及LevelDB环境的初始化等工作。

##### TabletIO

TabletIO是直接与LevelDB交互的模块，承接了TabletNode主要功能（加载、卸载、读、写、遍历等），下面重点看一下其成员变量。

```c++
mutable Mutex mutex_;              // 与db_ref_count_配合使用，是db_ref_count_的同步机制
TabletWriter* async_writer_;       // 负责异步写入
ScanContextManager* scan_context_manager_;  // 负责scan

std::string tablet_path_;          // tablet在dfs上的全路径（默认前缀是“../data/”）
const std::string start_key_;      // load tablet时 业务传入的start_key
const std::string end_key_;        // load tablet时 业务传入的end_key
const int64_t ctime_;              // load tablet时 业务传入的创建时间
const uint64_t version_;           // load tablet时 业务传入的版本号
const std::string short_path_;     // load tablet时 业务传入的（去除前缀的相对）路径
std::string raw_start_key_;        // 经过编码之后存储levledb的start_key_, 支持KV、ReadAble、Binary三种key类型
std::string raw_end_key_;          // 经过编码之后存储levledb的end_key_
CompactStatus compact_status_;     // tablet的压缩状态（kTableNotCompact、kTableOnCompact、kManualCompaction、kMinorCompaction、kTableCompacted）

TabletStatus status_;              // TabletNode层Tablet的状态
tera::TabletMeta::TabletStatus tablet_status_;  // LevelDB层Tablet的状态
std::string last_err_msg_;         // 最后一次load tablet是open db发生的错误
volatile int32_t ref_count_;       // tabletio 的引用次数，初始化为1，在外部调用是需要AddRef 和 DecRef，在DecRef 根据引用计数进行自动回收
volatile int32_t db_ref_count_;    // leveldb 的引用计数，在db_操作前后都会进行加减，与mutex_一起使用
leveldb::Options ldb_options_;     // leveldb的配置，详见后面文章的介绍
leveldb::DB* db_;                  // 代表leveldb
leveldb::Cache* m_memory_cache;    // 使用MemStore时有用，忽略
TableSchema table_schema_;         // load tablet时 业务传入的schema
bool kv_only_;                     // 是否是kv类型的tablet
std::map<uint64_t, uint64_t> id_to_snapshot_num_;   // map userid -> leveldb snap id
std::map<uint64_t, uint64_t> rollbacks_;            // leveldb snap id ->  rollback id， leveldb snap id从id_to_snapshot_num_获取

const leveldb::RawKeyOperator* key_operator_;   // key的操作抽象，支持KV、ReadAble、Binary三种key类型

std::map<std::string, uint32_t> cf_lg_map_;     // cf_name -> lg_id 映射
std::map<std::string, uint32_t> lg_id_map_;     // lg_name -> lg_id 映射

  // accept unload request for this tablet will inc this count
std::atomic<int> try_unload_count_;    // unload tablet 的计数，超过3次，认为是UrgentUnload
StatCounter counter_;                  // leveldb操作的统计信息，后续介绍
mutable Mutex schema_mutex_;           // schema 变更的锁
```

