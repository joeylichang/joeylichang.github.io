# 加载Tablet

TabletNode 加载 Tablet的主要内容包括：

1. leveldb option 初始化（levledb配置），详见（[tera_leveldb_options](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/tablet_node/leveldb/options.md)）。

2. leveldb open db。

3. TabletIo成员变量根据业务传入参数进行初始化。

   ### TabletIO::Load

   ```c++
   bool TabletIO::Load(const TableSchema& schema, const std::string& path,
                       const std::vector<uint64_t>& parent_tablets,
                       const std::set<std::string>& ignore_err_lgs, leveldb::Logger* logger,
                       leveldb::Cache* block_cache, leveldb::TableCache* table_cache,
                       StatusCode* status) {
     {
       MutexLock lock(&mutex_);
       if (status_ == kReady) {
         return true;
       } else if (status_ != kNotInit) {
         SetStatusCode(status_, status);
         return false;
       }
       status_ = kOnLoad;
       db_ref_count_++;    // 对db进行操作需要增加db的引用计数
     }
   
     // any type of table should have at least 1lg+1cf.
     table_schema_.CopyFrom(schema);
     if (table_schema_.locality_groups_size() == 0) {
       // only prepare for kv-only mode, no need to set fields of it.
       table_schema_.add_locality_groups();
     }
   
     RawKey raw_key = table_schema_.raw_key();
     if (raw_key == TTLKv || raw_key == GeneralKv) {
       kv_only_ = true;
     } else {
       // for compatible
       if (table_schema_.column_families_size() == 0) {
         // only prepare for kv-only mode, no need to set fields of it.
         table_schema_.add_column_families();
         kv_only_ = true;
       } else {
         kv_only_ = table_schema_.kv_only();
       }
     }
   
     // 如果是kv 则不能使用shardmemtable（levledb底层实现，见leveldb的讲解）
     if (kv_only_) {
       ldb_options_.memtable_shard_num = 0;
     } else {
       ldb_options_.memtable_shard_num = FLAGS_tera_leveldb_memtable_shard_num;
     }
   
     // 根据key的类型 ，选择相应的操作，后续介绍key的编码方式
     key_operator_ = GetRawKeyOperatorFromSchema(table_schema_);
     // [raw_start_key_, raw_end_key_)
     raw_start_key_ = start_key_;
     // 非kv进行编码start_key_，变成leveldb存储的raw_start_key_
     if (!kv_only_ && !start_key_.empty()) {
       key_operator_->EncodeTeraKey(start_key_, "", "", kLatestTs, leveldb::TKT_FORSEEK,
                                    &raw_start_key_);
       // kv进行编码start_key_，变成leveldb存储的raw_start_key_
     } else if (kv_only_ && table_schema_.raw_key() == TTLKv && !start_key_.empty()) {
       key_operator_->EncodeTeraKey(start_key_, "", "", 0, leveldb::TKT_FORSEEK, &raw_start_key_);
     }
     // end_key_编码 与 start_key_相同
     raw_end_key_ = end_key_;
     if (!kv_only_ && !end_key_.empty()) {
       key_operator_->EncodeTeraKey(end_key_, "", "", kLatestTs, leveldb::TKT_FORSEEK, &raw_end_key_);
     } else if (kv_only_ && table_schema_.raw_key() == TTLKv && !end_key_.empty()) {
       key_operator_->EncodeTeraKey(end_key_, "", "", 0, leveldb::TKT_FORSEEK, &raw_end_key_);
     }
   
     // levledb options的设置，详见tera_leveldb_options的介绍
     ldb_options_.key_start = raw_start_key_;
     ldb_options_.key_end = raw_end_key_;
     ldb_options_.l0_slowdown_writes_trigger = FLAGS_tera_tablet_level0_file_limit;
     ldb_options_.max_sub_parallel_compaction = FLAGS_tera_tablet_max_sub_parallel_compaction;
     ldb_options_.ttl_percentage = FLAGS_tera_tablet_ttl_percentage;
     ldb_options_.del_percentage = FLAGS_tera_tablet_del_percentage;
     ldb_options_.block_size = FLAGS_tera_tablet_write_block_size * 1024;
     ldb_options_.max_block_log_number = FLAGS_tera_tablet_max_block_log_number;
     ldb_options_.write_log_time_out = FLAGS_tera_tablet_write_log_time_out;
     ldb_options_.log_async_mode = FLAGS_tera_log_async_mode;
     ldb_options_.info_log = logger;
     ldb_options_.max_open_files = FLAGS_tera_memenv_table_cache_size;
     ldb_options_.manifest_switch_size = FLAGS_tera_leveldb_manifest_switch_size_MB;
     ldb_options_.max_background_compactions = FLAGS_tera_leveldb_max_background_compactions;
     ldb_options_.slow_down_level0_score_limit = FLAGS_tera_leveldb_slow_down_level0_score_limit;
     ldb_options_.ignore_corruption_in_open = FLAGS_tera_leveldb_ignore_corruption_in_open;
   
     ldb_options_.use_memtable_on_leveldb = FLAGS_tera_tablet_use_memtable_on_leveldb;
     ldb_options_.memtable_ldb_write_buffer_size =
         FLAGS_tera_tablet_memtable_ldb_write_buffer_size * 1024;
     ldb_options_.memtable_ldb_block_size = FLAGS_tera_tablet_memtable_ldb_block_size * 1024;
     if (FLAGS_tera_tablet_use_memtable_on_leveldb) {
       LOG(INFO) << "enable mem-ldb for this tablet-server:"
                 << " buffer_size:" << ldb_options_.memtable_ldb_write_buffer_size
                 << ", block_size:" << ldb_options_.memtable_ldb_block_size;
     }
   
     uint32_t bloom_filter_bits_per_key = table_schema_.has_bloom_filter_bits_per_key()
                                              ? table_schema_.bloom_filter_bits_per_key()
                                              : 10;
   
     LOG(INFO) << "Use " << bloom_filter_bits_per_key << " bits per key for bloom filter.";
   
     // 根据key 类型设置filter
     if (kv_only_ && table_schema_.raw_key() == TTLKv) {
       ldb_options_.filter_policy = leveldb::NewTTLKvBloomFilterPolicy(bloom_filter_bits_per_key);
     } else if (kv_only_) {
       ldb_options_.filter_policy = leveldb::NewBloomFilterPolicy(bloom_filter_bits_per_key);
     } else if (table_schema_.raw_key() == Readable) {
       ldb_options_.filter_policy = leveldb::NewRowKeyBloomFilterPolicy(
           bloom_filter_bits_per_key, leveldb::ReadableRawKeyOperator());
     } else {
       CHECK_EQ(table_schema_.raw_key(), Binary);
       ldb_options_.filter_policy = leveldb::NewRowKeyBloomFilterPolicy(
           bloom_filter_bits_per_key, leveldb::BinaryRawKeyOperator());
     }
     ldb_options_.block_cache = block_cache;
     ldb_options_.table_cache = table_cache;
     ldb_options_.flush_triggered_log_num = FLAGS_tera_tablet_flush_log_num;
     ldb_options_.log_file_size = FLAGS_tera_tablet_log_file_size * 1024 * 1024;
     ldb_options_.parent_tablets = parent_tablets;
     if (table_schema_.raw_key() == Binary) {
       ldb_options_.raw_key_format = leveldb::kBinary;
       ldb_options_.comparator = leveldb::TeraBinaryComparator();
     } else if (table_schema_.raw_key() == TTLKv) {  // KV-Pair-With-TTL
       ldb_options_.raw_key_format = leveldb::kTTLKv;
       ldb_options_.comparator = leveldb::TeraTTLKvComparator();
       ldb_options_.enable_strategy_when_get = true;  // active usage of strategy in DB::Get
     } else {                                         // Readable-Table && KV-Pair-Without-TTL
       ldb_options_.raw_key_format = leveldb::kReadable;
       ldb_options_.comparator = leveldb::BytewiseComparator();
     }
     ldb_options_.verify_checksums_in_compaction = FLAGS_tera_leveldb_verify_checksums;
     ldb_options_.ignore_corruption_in_compaction = FLAGS_tera_leveldb_ignore_corruption_in_compaction;
     ldb_options_.use_file_lock = FLAGS_tera_leveldb_use_file_lock;
     ldb_options_.disable_wal = table_schema_.disable_wal();
     
     // lg 既DBImpl的option 设置，详见tera_leveldb_options的介绍
     SetupOptionsForLG(ignore_err_lgs);
   
     std::string path_prefix = FLAGS_tera_tabletnode_path_prefix;
     if (*path_prefix.rbegin() != '/') {
       path_prefix.push_back('/');
     }
   
     tablet_path_ = path_prefix + path;
     ldb_options_.dfs_storage_path_prefix = path_prefix;
     LOG(INFO) << "[Load] Start Open " << tablet_path_ << ", kv_only " << kv_only_
               << ", raw_key_operator " << key_operator_->Name();
   
     // open db（load的最终目的）
     leveldb::Status db_status = leveldb::DB::Open(ldb_options_, tablet_path_, &db_);
   
     if (!db_status.ok()) {
       LOG(ERROR) << "fail to open table: " << tablet_path_ << ", " << db_status.ToString();
       {
         MutexLock lock(&mutex_);
         status_ = kNotInit;
         last_err_msg_ = db_status.ToString();
         db_ref_count_--;
       }
       SetStatusCode(db_status, status);
       //         delete ldb_options_.env;
       return false;
     }
   
     async_writer_ = new TabletWriter(this);
     async_writer_->Start();
   
     scan_context_manager_ = new ScanContextManager;
   
     {
       MutexLock lock(&mutex_);
       status_ = kReady;
       // reset try unload count to 0 for ready
       // reset try_unload_count_ 唯一的地方，unload会一直++
       try_unload_count_ = 0;
       
       // 对db操作完毕，对计数进行减减
       db_ref_count_--;
     }
   
     LOG(INFO) << "[Load] Load " << tablet_path_ << " done";
     return true;
   }
   ```

   下面看一下lg（DBIpml）配置的初始化：

   ```c++
   void TabletIO::SetupOptionsForLG(const std::set<std::string>& ignore_err_lgs) {
     if (kv_only_) {
       if (RawKeyType() == TTLKv) {
         ldb_options_.compact_strategy_factory = new KvCompactStrategyFactory(table_schema_);
       } else {
         ldb_options_.compact_strategy_factory = new leveldb::DummyCompactStrategyFactory();
       }
     } else if (FLAGS_tera_leveldb_compact_strategy == "default") {
       // default strategy
       ldb_options_.compact_strategy_factory = new DefaultCompactStrategyFactory(table_schema_);
     } else {
       ldb_options_.compact_strategy_factory = new leveldb::DummyCompactStrategyFactory();
     }
   
     // exist_lg_list 记录所有的lg id
     std::set<uint32_t>* exist_lg_list = new std::set<uint32_t>;
     // lg_info_list 是 lg_id -> lg ottion的映射
     std::map<uint32_t, leveldb::LG_info*>* lg_info_list = new std::map<uint32_t, leveldb::LG_info*>;
     std::set<uint32_t> ignore_corruption_in_open_lg_list;
   
     int64_t triggered_log_size = 0;
     for (int32_t lg_i = 0; lg_i < table_schema_.locality_groups_size(); ++lg_i) {
       if (table_schema_.locality_groups(lg_i).is_del()) {
         continue;
       }
       const LocalityGroupSchema& lg_schema = table_schema_.locality_groups(lg_i);
       bool compress = lg_schema.compress_type();
       StoreMedium store = lg_schema.store_type();
   
       leveldb::LG_info* lg_info = new leveldb::LG_info(lg_schema.id());
       lg_info->memtable_shard_num = ldb_options_.memtable_shard_num;
   
       // 环境的设置，在这里只关注 DFS（SDFS）的情况
       // 既 最后一种情况的 lg_info->env = LeveldbBaseEnv();
       // LeveldbBaseEnv 会获取HDFS的环境，在TableNodeIpml中根据配置对HDFS进行了初始化
       if (mock_env_ != NULL) {
         // for testing
         LOG(INFO) << "mock env used";
         lg_info->env = LeveldbMockEnv();
       } else if (store == MemoryStore) {
         if (FLAGS_tera_use_flash_for_memenv) {
           if (!GetCachePaths().empty()) {
             if (FLAGS_tera_enable_persistent_cache) {
               lg_info->env = LeveldbBaseEnv();
               auto s = GetPersistentCache(&lg_info->persistent_cache);
               assert(s.ok());
             } else {
               lg_info->env = LeveldbFlashEnv();
               if (!lg_info->env) {
                 lg_info->env = LeveldbBaseEnv();
               }
             }
           }
         } else {
           lg_info->env = LeveldbMemEnv();
         }
         lg_info->seek_latency = 0;
         lg_info->block_cache = m_memory_cache;
       } else if (store == FlashStore && !GetCachePaths().empty()) {
         if (FLAGS_tera_enable_persistent_cache) {
           lg_info->env = LeveldbBaseEnv();
           auto s = GetPersistentCache(&lg_info->persistent_cache);
           assert(s.ok());
         } else {
           lg_info->env = LeveldbFlashEnv();
           if (!lg_info->env) {
             lg_info->env = LeveldbBaseEnv();
           } else {
             lg_info->use_direct_io_read = FLAGS_tera_leveldb_use_direct_io_read;
             lg_info->use_direct_io_write = FLAGS_tera_leveldb_use_direct_io_write;
             lg_info->posix_write_buffer_size = FLAGS_tera_leveldb_posix_write_buffer_size;
           }
         }
         if (lg_info->persistent_cache || lg_info->env == LeveldbFlashEnv()) {
           lg_info->seek_latency = FLAGS_tera_leveldb_env_local_seek_latency;
         }
       } else {
         lg_info->env = LeveldbBaseEnv();
         lg_info->seek_latency = FLAGS_tera_leveldb_env_dfs_seek_latency;
       }
   
       if (compress) {
         lg_info->compression = leveldb::kSnappyCompression;
       }
   
       lg_info->block_size = lg_schema.block_size() * 1024;
       if (lg_schema.use_memtable_on_leveldb()) {
         lg_info->use_memtable_on_leveldb = true;
         lg_info->memtable_ldb_write_buffer_size = lg_schema.memtable_ldb_write_buffer_size() * 1024;
         lg_info->memtable_ldb_block_size = lg_schema.memtable_ldb_block_size() * 1024;
         LOG(INFO) << "enable mem-ldb for LG:" << lg_schema.name().c_str()
                   << ", buffer_size:" << lg_info->memtable_ldb_write_buffer_size
                   << ", block_size:" << lg_info->memtable_ldb_block_size;
       }
       lg_info->sst_size = lg_schema.sst_size();
       // FLAGS_tera_tablet_write_buffer_size is the max buffer size
       int64_t max_size = FLAGS_tera_tablet_max_write_buffer_size * 1024 * 1024;
       if (lg_schema.sst_size() * 4 < max_size) {
         lg_info->write_buffer_size = lg_schema.sst_size() * 4;
       } else {
         lg_info->write_buffer_size = max_size;
       }
       triggered_log_size += lg_info->write_buffer_size;
       lg_info->table_builder_batch_write = (FLAGS_tera_leveldb_table_builder_write_batch_size > 0);
       lg_info->table_builder_batch_size = FLAGS_tera_leveldb_table_builder_write_batch_size;
       exist_lg_list->insert(lg_i);
       (*lg_info_list)[lg_i] = lg_info;
       if (ignore_err_lgs.find(lg_schema.name()) != ignore_err_lgs.end()) {
         ignore_corruption_in_open_lg_list.insert(lg_i);
       }
     }
     if (mock_env_ != NULL) {
       ldb_options_.env = LeveldbMockEnv();
     } else {
       ldb_options_.env = LeveldbBaseEnv();
     }
   
     if (exist_lg_list->size() == 0) {
       delete exist_lg_list;
     } else {
       ldb_options_.exist_lg_list = exist_lg_list;
       ldb_options_.flush_triggered_log_size = triggered_log_size * 2;
     }
     if (lg_info_list->size() == 0) {
       delete lg_info_list;
     } else {
       ldb_options_.lg_info_list = lg_info_list;
       ldb_options_.ignore_corruption_in_open_lg_list = ignore_corruption_in_open_lg_list;
     }
   
     // 根据lg 和 cf信息对cf_lg_map_ 和 lg_id_map_进行初始化，详见TabletNode的类架构部分TabletIO的成员变量介绍部分
     IndexingCfToLG();
   }
   ```

   

### Key 编码格式

```c++
enum TeraKeyType {
  TKT_FORSEEK = 0,
  TKT_DEL = 1,
  TKT_DEL_COLUMN = 2,
  TKT_DEL_QUALIFIERS = 3,
  TKT_DEL_QUALIFIER = 4,
  TKT_VALUE = 5,
  // 6 is reserved, do not use
  TKT_ADD = 7,
  TKT_PUT_IFABSENT = 8,
  TKT_APPEND = 9,
  TKT_ADDINT64 = 10,
  TKT_TYPE_NUM = 11
};

/**
 *  readable encoding format:
 *  [rowkey\0|column\0|qualifier\0|type|timestamp]
 *  [ rlen+1B| clen+1B| qlen+1B   | 1B | 7B      ]
 **/

/**
 *  binary encoding format:
 *  [rowkey|column\0|qualifier|type|timestamp|rlen|qlen]
 *  [ rlen | clen+1B| qlen    | 1B |   7B    | 2B | 2B ]
 **/

// support KV-pair with TTL, Key's format :
// [row_key|expire_timestamp]
// [rlen|8B]

```

