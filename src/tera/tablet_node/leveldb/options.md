# LevelDB 配置

在对LevelDB的有了了解之后，有必要了解一下其配置，有以下两个好处：

1. 性能优化：有了整体架构和细节的了解之后，可以通过配置对性能进行调优。
2. 线上问题定位：Tera有很多统计数据，可以帮助定位和解决线上运行时遇到的问题，很多配置与这些统计相关。



### DBTable 配置（Tablet）

##### Tera新增配置

###### 可配置

1. use_memtable_on_leveldb

   默认为false，透传给lg（DBImpl）（详情见leveldb介绍）。

2. memtable_ldb_write_buffer_size

   如果设置use_memtable_on_leveldb，透传给lg（DBImpl），可在schema中设置大小（schema的默认值是1M）。用于创建lg（DBImpl）实例，透传给write_buffer_size（LeavelDB原有配置），详见后面DBImpl介绍。DBTable中该值默认配置也是1M（如果不设定use_memtable_on_leveldb，好像无用）。

3. memtable_ldb_block_size

   如果设置use_memtable_on_leveldb，透传给lg（DBImpl），可在schema中设置大小（schema的默认值是4K）。用于创建lg（DBImpl）实例，透传给block_size（LeavelDB原有配置），详见后面DBImpl介绍。DBTable中该值默认配置也是4K（如果不设定use_memtable_on_leveldb，好像无用）。

4. verify_checksums_in_compaction

   默认为true，透传给ReadOptions，在ParseBlock时对数据和类型进行crc32c校验。

5. ignore_corruption_in_compaction

   默认false，压缩过程是否忽略错误，有一场直接Abandon。

6. use_file_lock

   levelds的HDFS环境不支持。

7. ignore_corruption_in_open

   默认是false，在open db是否忽略数据异常的问题。

8. ttl_percentage

   默认值99，sst文件触发压缩的闸值，超过站址的文件会加入带压缩队列。

9. del_percentage

   默认值20，sst文件触发压缩的闸值，超过站址的文件会加入带压缩队列。

10. max_background_compactions

   默认值8，压缩的线程数。

11. slow_down_level0_score_limit

    默认值100，一种level 0层压缩打分降级的策略。

    level 0：如果 level 0文件数小于等于slow_down_level0_score_limit，score =  level 0文件数 / 2

    ​               如果 level 0文件数大于slow_down_level0_score_limit，score =  （level 0文件数 / 2）的开方。

    注意：目的是减小level 0 的分数，从而减少level 0 的合并。

    1. mumentable 过大减少压缩。
    2. 如果level 0 层很多小文件（批量的删除或者覆盖）文件较多（字节数不多不能够压缩，使用文件为依据则可以压缩）也会影响查询性能。
    3. 如果level 0 文件较多，也可能是写入较多，此时应比较压缩防止影响性能。

    levle 1：score = level_bytes / 5 * sst_size。

    level 2 ：score = level_bytes / 50 * sst_size。……以此类推（默认7层）。

12. max_sub_parallel_compaction

    默认值10，max_background_compactions 的基础上可以并行的自压缩的数量。

13. memtable_shard_num

    透传给lg（DBImpl），见lg（DBImpl）介绍。

14. l0_slowdown_writes_trigger

    默认值20，判断leveldb 是否处于Busy的闸值。

15. max_block_log_number

    默认值50，log文件数量的闸值（未消费完），找过闸值不可以switch log。

16. write_log_time_out

    默认值5，写log 和 sync log的超时时间，失败会循环重试每次左移1，中间有一些switch_log的逻辑（包括等log消费完等）。

17. log_async_mode

    默认为true，是否支持异步写log。

18. manifest_switch_size

    默认2M，mamfeist切换的闸值，除了大小还有时间间隔见manifest_switch_interval，两个标准是并的关系。

19. flush_triggered_log_num

    默认值10w，好像未使用。

20. log_file_size

    默认值32M，log文件的大小闸值。

###### 不可配置

1. key_start

   请求中带的参数。

2. key_end

   请求中带的参数。

3. raw_key_format

   支持三种格式：kBinary、kTTLKv、kReadable

4. parent_tablets

   请求中带的参数。

5. persistent_cache

   见之前文件介绍。

6. disable_wal

   默认值false，table_schema_指定的参数。是否禁止写log。

   主要：tablet公用一个log，既所有的lg公用。

7. dfs_storage_path_prefix

   默认值"../data/"，dfs的路径前缀。

8. enable_strategy_when_get

   只针对TTLKV 类型的rawkey，为true，其余为false。

   删除TableCahe中数据时，是否使用压缩策略。

9. compact_strategy_factory

   TTLKv：KvCompactStrategyFactory

   GeneralKv：DummyCompactStrategyFactory

   其余类型，使用default（可配置）：DefaultCompactStrategyFactory

   其他配置：DummyCompactStrategyFactory

10. exist_lg_list

    记录lg的id的set类型结构，DBTable内部遍历lg 的时候使用（配合lg_info_list遍历lg）。

11. flush_triggered_log_size

    所有lg的write_buffer_size 的2倍。

    写操作时，遍历lg判断判断更新之后的数据是否超过闸值，是的话MakeRoom。

    注意：这个闸值时DBTable，检查的是lg的DBImpl，所以是一个上限check。

12. lg_info_list

    LG_info的map，是lg的options。

13. ignore_corruption_in_open_lg_list

    参数传递进来，代表lg忽略open时的异常。

14. seek_latency （默认值10000000）

    透传给lg（DBImpl）。

15. dump_mem_on_shutdown（默认值 true）

    关闭db时，是否dump内存的数据。

16. drop_base_level_del_in_compaction（默认值true）

17. sst_size（默认值kDefaultSstSize）

    透传给lg（DBImpl）。

18. use_direct_io_read（默认值false）

    dfs忽略。

19. use_direct_io_write（默认值false）

    dfs忽略。

20. posix_write_buffer_size（默认值512K）

    dfs忽略。

21. table_builder_batch_write （默认值false）

    透传给lg（DBImpl），见lg（DBImpl）介绍。

22. table_builder_batch_size（默认值0）

    透传给lg（DBImpl），见lg（DBImpl）介绍。

23. manifest_switch_interval（默认值600）

    600s = 10min，mamfeist切换的闸值，同事大小要超过2M。

##### LeavelDB原有配置

###### 可配置

1. block_size

   DBTable未使用，在DBImpl中有使用。

2. max_open_files（好像是bug呢？）

###### 不可配置

1. info_log

   leveldb的运行日志，不是wal，详细配置见后面”本地Env配置“。

2. filter_policy

   格式不用处理方式不同。

   raw_key() == TTLKv -> NewTTLKvBloomFilterPolicy

   raw_key() == GeneralKv -> NewBloomFilterPolicy

   raw_key() == Readable -> NewRowKeyBloomFilterPolicy

   schema_.raw_key(), == Binary -> NewRowKeyBloomFilterPolicy

3. block_cache

   透传给lg（DBImpl），见lg（DBImpl）介绍。

4. table_cache

   默认配置2G，所有的lg（DBImpl）共用。

5. comparator 

   不同格式key级别的操作，每种key的组织形式不一样，比较也不一样。

   raw_key_format = leveldb::kBinary -> TeraBinaryComparator

   raw_key_format = leveldb::kTTLKv -> TeraTTLKvComparator

   raw_key_format = leveldb::kReadable -> BytewiseComparator

6. env

   关注hdfs环境。

7. error_if_exists（默认值false）

8. paranoid_checks（默认值false）

9. write_buffer_size（默认值4MB）

   DBTable未使用，在DBImpl中有使用。

10. block_restart_interval（默认值16）

    block 压缩key的闸值。

11. compression（默认值kSnappyCompression）

    DBTable未使用，见DBIpml使用。



### DBImpl配置（LocalityGroups）

##### Tera新增配置

1. seek_latency

   dfs环境默认1kw。对一个文件seek到一定程度就进行压缩，之间的计算方式压缩成本与读取和写入的成本之间的计算，如果压缩 对应25次读写的成本，那么压缩就是划算的。

   计算公式：

   size_for_one_seek = 16384ULL * vset_->options_->seek_latency / 10000000; （默认16K seek一次）

   f->allowed_seeks = (f->file_size / size_for_one_seek);（文件被seek的次数）

2. use_memtable_on_leveldb

   通过schema设置，默认为false。

3. memtable_ldb_write_buffer_size

   见DBTable介绍

4. memtable_ldb_block_size

   见DBTable介绍

5. sst_size

   通过lg_schema指定，默认值8M，不能超过32M。如果sst_size * 4 < 32M，write_buffer_size = sst_size * 4。

6. use_direct_io_read

   dfs忽略。

7. use_direct_io_write

   dfs忽略。

8. posix_write_buffer_size

   dfs忽略。

9. table_builder_batch_write

   tera配置为true，Table（内存的kv map，immutable的数据结构），在内存中缓存的数据。

10. table_builder_batch_size

    tera配置为256K，Table（内存的kv map，immutable的数据结构），在内存中缓存的数据。

11. persistent_cache

    见之间文章介绍。

12. memtable_shard_num

    MemTable（ShardedMemTable）分片的数量，默认值是4。如果是KV结构则，不使用ShardedMemTable，而是MemTableOnLevelDB或者BaseMemTable，配置是0。KV需要Get接口，其他接口时通过迭代器获取（因为ShardedMemTable组装了多个TableCache）数据。所以ShardedMemTable，不支持Get。MemTableOnLevelDB 和 BaseMemTable 支持Get接口。

##### LeavelDB原有配置

1. env：在这里只关注HDFS。

2. block_cache 

   默认配置2G，所有的lg（DBImpl）共用。

3. compression

   kSnappyCompression，lg_schema指定。

4. write_buffer_size

   在 64K 到 1G之间，如果不在范围内，小于 64K 取 64K，大于 1G 取 1G。

   mumemtable的大小。

5. block_size

   默认4K，可在schema中指定。在 1K 到  4M之间，如果不在范围内，小于1K取1K，大于 4M取4M。

   块缓存中一块的大小，注意是用户数据的真实大小，如果启动压缩读取的磁盘数据会小于这个值。



### 本地Env配置（Posix）

NewLogger(FLAGS_tera_leveldb_log_path, log_opt, &ldb_logger_);_

路径：./log/leveldb.log。

LogOption.SetMaxLogSize(max_log_size)

size：1G。

LogOption.SetFlushTriggerSize(FLAGS_leveldb_log_flush_trigger_size_B)

flush：1M。

LogOption.SetFlushTriggerIntervalMs(FLAGS_leveldb_log_flush_trigger_interval_ms)

interval：1s，与FlushTriggerSize 满足其一即可。

leveldb::Env::Default()->SetLogger(ldb_logger_)

