

# Volume 接口梳理

#### 启动参数

| option                 | default                                   | description                                                  |
| ---------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| port                   | 8080                                      | http listen port                                             |
| port.public            | 0                                         | port opened to public                                        |
| ip                     |                                           | ip or server name                                            |
| publicUrl              | ip:port                                   |                                                              |
| ip.bind                |                                           | ip address to bind to                                        |
| mserver                |                                           | comma-separated master servers                               |
| pulseSeconds           | 5                                         | number of seconds between heartbeats                         |
| idleTimeout            | 30                                        | connection idle seconds                                      |
| dataCenter             |                                           |                                                              |
| rack                   |                                           |                                                              |
| index                  | memory/leveldb/leveldbMedium/leveldbLarge | memory~performance balance                                   |
| images.fix.orientation | false                                     | Adjust jpg orientation when uploading                        |
| read.redirect          | true                                      | Redirect moved or non-local volumes                          |
| cpuprofile             | ""                                        | cpu profile output file                                      |
| memprofile             | ""                                        | memory profile output file                                   |
| compactionMBps         | 0                                         | limit background compaction or copying speed in mega bytes per second |
| dir                    |                                           | directories to store data files. dir[,dir]...                |
| max                    | 7                                         | maximum numbers of volumes, count[,count]...                 |
| whiteList              |                                           | No limit if empty                                            |

#### http请求

| path           | function                           | note          |
| -------------- | ---------------------------------- | ------------- |
| /ui/index.html |                                    |               |
| /status        |                                    |               |
| /stats/counter |                                    |               |
| /stats/memory  |                                    |               |
| /stats/disk    |                                    |               |
| /              | "GET", "HEAD" ==> GetOrHeadHandler | ReadRequest   |
| /              | "DELETE" ==> DeleteHandler         | DeleteRequest |
| /              | "PUT", "POST" ==> PostHandler      | WriteRequest  |



#### grpc请求

| method_name           | descriptiom                          |
| --------------------- | ------------------------------------ |
| BatchDelete           | 批量删除volume（weed shell使用）     |
| VacuumVolumeCheck     | 压缩检查                             |
| VacuumVolumeCompact   | 压缩执行                             |
| VacuumVolumeCommit    | 压缩提交                             |
| VacuumVolumeCleanup   | 压缩失败清理中间数据                 |
| DeleteCollection      | 删除collection（weed shell使用）     |
| AllocateVolume        | 申请volume（master 调用）            |
| VolumeSyncStatus      | 同步volume状态信息                   |
| VolumeIncrementalCopy | weed shell使用，见weed shell部分介绍 |
| VolumeMount           | weed shell使用，见weed shell部分介绍 |
| VolumeUnmount         | weed shell使用，见weed shell部分介绍 |
| VolumeDelete          | 删除volume                           |
| VolumeMarkReadonly    | 标志volume只读                       |
| VolumeCopy            | weed shell使用，见weed shell部分介绍 |
| ReadVolumeFileStatus  | 获取volume 数据文件和索引文件信息    |
| CopyFile              | 复制文件（weed shell使用，后续介绍） |
| VolumeTailSender      | weed shell使用，见weed shell部分介绍 |
| VolumeTailReceiver    | weed shell使用，见weed shell部分介绍 |
| Query                 | 查询碧辟fid信息                      |

