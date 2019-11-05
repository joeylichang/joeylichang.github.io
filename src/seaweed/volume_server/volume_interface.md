

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

| method_name           | descriptiom |
| --------------------- | ----------- |
| BatchDelete           |             |
| VacuumVolumeCheck     |             |
| VacuumVolumeCompact   |             |
| VacuumVolumeCommit    |             |
| VacuumVolumeCleanup   |             |
| DeleteCollection      |             |
| AllocateVolume        |             |
| VolumeSyncStatus      |             |
| VolumeIncrementalCopy |             |
| VolumeMount           |             |
| VolumeUnmount         |             |
| VolumeDelete          |             |
| VolumeMarkReadonly    |             |
| VolumeCopy            |             |
| ReadVolumeFileStatus  |             |
| CopyFile              |             |
| VolumeTailSender      |             |
| VolumeTailReceiver    |             |
| Query                 |             |

