6Master

##### 启动参数

| option                  | default      | description                            |
| ----------------------- | ------------ | -------------------------------------- |
| port                    | 9333         | http listen port                       |
| ip                      | localhost    | master <ip>、<server> address          |
| ip.bind                 | 0.0.0.0      | ip address to bind to                  |
| mdir                    | os.TempDir() | data directory to store meta data      |
| peers                   | ""           | example: 127.0.0.1:9093,127.0.0.1:9094 |
| volumeSizeLimitMB       | 30*1000      |                                        |
| volumePreallocate       | false        | Preallocate disk space for volumes     |
| pulseSeconds            | 5            | number of seconds between heartbeats   |
| defaultReplication      | 000          |                                        |
| garbageThreshold        | 0.3          |                                        |
| whiteList               | ""           |                                        |
| disableHttp             | false        | disable http requests, only gRPC       |
| metrics.address         |              |                                        |
| metrics.intervalSeconds | 15           |                                        |



##### http请求

| path            | function                                              | note               |
| --------------- | ----------------------------------------------------- | ------------------ |
| /               | 路由给主，展示主的信息（拓扑、请求统计、容量统计）    | all                |
| /ui/index.html  | 信息展示，展示当前master的信息（信息如上）            | all                |
| /dir/assign     | 申请fid                                               | global             |
| /dir/lookup     | 查找volumeId的loction                                 | global             |
| /dir/status     | 查询拓扑信息                                          | global             |
| /col/delete     | 删除collection                                        | collection         |
| /vol/grow       | 申请volume                                            | volume             |
| /vol/status     | 全部volume的信息                                      | volume             |
| /vol/vacuum     | 开启一次gc，可以指定gc的闸值                          | volume             |
| /submit         | 合并申请fid 和 写入volume_server 两个请求为一个       | 不需要转发给leader |
| /stats/health   | 探测消息，返回version                                 | 不需要转发给leader |
| /stats/counter  | 请求统计（Connections、Requests、读写删除、字节大小） | 不需要转发给leader |
| /stats/memory   | 内存统计                                              | 不需要转发给leader |
| "/{fileId}      | 请求转发给volume_server                               | 不需要转发给leader |
| /cluster/status | raft集群的信息                                        | 不需要转发给leader |



##### grpc请求

| method_name            | descriptiom                                            |
| ---------------------- | ------------------------------------------------------ |
| SendHeartbeat          | 心跳，创建dc、rack、node、volume、删除volume           |
| KeepConnected          | 客户端创建一个流，实时观察volume的位置                 |
| LookupVolume           | 指定vid、collection查看拓扑                            |
| Assign                 | 分配fid                                                |
| Statistics             | 指定collection 通知容量TotalSize、UsedSize、FileCount  |
| CollectionList         | collection查询（名字）                                 |
| CollectionDelete       | 删除collection                                         |
| VolumeList             | volume的统计信息，VolumeCount、MaxVolumeCount等        |
| LookupEcVolume         |                                                        |
| GetMasterConfiguration | 普罗米修斯信息，MetricsAddress、MetricsIntervalSeconds |

