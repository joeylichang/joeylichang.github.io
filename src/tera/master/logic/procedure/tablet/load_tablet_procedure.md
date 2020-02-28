# Master Load Tablet 流程

Master侧load tablet的流程中状态的转换不是内部自定的，与tablet的状态相关。

1. 如果是kTabletReady 或者 kTabletLoadFail ，则load直接结束。
2. 如果是kTabletOffline 或者 kTabletDelayOffline。
   1. 如果节点down。
      1. 如果是meta_table，转kTsOffline。
      2. 如果不是meta_table，并且在内存中有节点数据，则转kTsRestart。
      3. 如果不是meta_table，并且在内存中没有节点数据，且目标节点为kPendingOffline，则转kTsDelayOffline。
      4. 如果不是meta_table，并且在内存中没有节点数据，且目标节点不是kPendingOffline，则转kTsOffline。
   2. 如果是meta_table，则转kLoadTablet。
   3. 如果是不是meta_table，且没有更新元数据，则转kUpdateMeta。
   4. 如果不是meta_yable，且更新完元数据，并且节点load并发不超过20个，则转kLoadTablet。
   5. 如果不是meta_yable，且更新完元数据，并且节点load并发超过20个，kTsLoadBusy
3. 如果是kTabletLoading。
   1. 如果超过重试次数（默认5次），转kTsLoadFail。
   2. 如果没有超过重试次数，并且节点down，重复2.1逻辑。
   3. 如果没有超过重试次数，且节点没有down，且load请求已发但是没有返回，则转kWaitRpcResponse。
   4. 如果如果没有超过重试次数，且节点没有down，且load请求已返回，转kTsLoadSucc。



