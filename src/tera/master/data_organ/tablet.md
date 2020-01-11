# Tablet

```c++
class Tablet {
  protected:
  	mutable Mutex mutex_;               // tablet 变量修改的锁
  	TabletMeta meta_;                   // pb结构，描述tablete，后面介绍
  	TabletStateMachine state_machine_;  // tablet 状态转换机，后面介绍
  private:
  	TabletNodePtr node_;                // tablet 归属的节点
    TablePtr table_;                    // tablet 归属的table
    int64_t update_time_;               // tablet 更新时间
    int64_t last_move_time_us_;         // tablet 最后移动的时间
    int64_t data_size_on_flash_;        // tablet 在flash和memory上的总大小
    std::string server_id_;             // 未使用

    std::vector<std::string> ignore_err_lgs_; // lg array for ignore_err_, 一个lg一个元素
    std::list<TabletCounter> counter_list_;   // 未用
    TabletCounter average_counter_;           // 统计信息，当前周期权重2份，前一周期1份（见pb）
    void* merge_param_;                       // 未用						

    // Tablet Split History Tracing
    struct TabletSplitHistory {
      int64_t last_split_ts;

      TabletSplitHistory() : last_split_ts(0) {}
    } split_history_;                        // spilt间隔10min

    bool in_transition_ = false;             // 事务是否在进行的标志

    // protected by Table::mutex_
    bool gc_reported_;                       // table中会调用，见后续介绍
    std::multiset<TabletFile> inh_files_;    // table中合并tablet会用到，建后续介绍

    // sucessive load failed count, will be cleared on tablet load succeed
    std::atomic<int> load_fail_cnt_;         // 加载失败统计
    const int64_t create_time_;              // 创建时间
}
```



### TabletMeta

TabletMeta是一个pb结构是系统内交互的关于tablet的数据。

```protobuf
message TabletMeta {
  required string table_name = 1;     // tablet 所属table的名字
  required string path = 2;           // 路径
  required KeyRange key_range = 3;    // key 范围
  optional string server_addr = 4;    // tablet node 地址
  optional TabletStatus status = 5;   // tablet 状态，详见下面的转换图
  optional int64 size = 6;            // tablet 大小
  optional bool compress = 7;         // 是否压缩
  optional CompactStatus compact_status = 8 [default = kTableNotCompact]; // 不压缩、压缩ing、压缩完成
  optional StoreMedium store_medium = 9   // flash/memory/disk
      [default = DiskStore];  // for Compatible
  repeated uint64 parent_tablets = 12;    // 分裂使用
  repeated int64 lg_size = 13;            // lg数组
  optional int64 last_move_time_us = 15;  // 上次移动时间
  optional int64 data_size_on_flash = 16; // 在flash、memory上面的总大小
  optional int64 create_time = 17;        // 创建时间
  optional uint64 version = 18;           // 版本号
}
```



### Tablet状态转换图

![tera_tablet_state_change](../../../../images/tera_tablet_state_change.png)

1. 正常的生命周期：offline -> loading -> ready -> unloading -> offline。
2. 特殊状态：
   1. 延期下架 delayoffline：在offline、loading、ready可以延期下架。
   2. 失效状态 disable：在offline 和 loadfaile是可以进行disable。
3. splitted、mergerd两个状态，没有了后续的转换，这也是一个比较奇怪的点？
