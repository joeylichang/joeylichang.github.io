# CockroachDB KV层介绍

## 单机存储引擎
CockroachDB单机存储引擎使用的是RocksDB，RocksDB是Facebook开源基于Google LevelDB的改进项目。RocksDB相比LevelDB主要改进了一下几方面：
1. 支持多线程Compaction，LevelDB是单线程Compaction。
2. 支持配置多个MemTable，Level DB两个MemTable在写请求量较大时可能会堵塞。
3. 支持多种压缩算法，包括zlib、bzip2、snappy（LevelDB）等。

接下来介绍一下RocksDB的原理，后续会有专栏介绍。

RocksDB在引擎的数据模型上沿用了LevelDB的LSM，将随机写转化为顺序写提高了写入性能，牺牲读性能。主要步骤如下：
1. 写操作先将数据追加写到磁盘WAL（write head log）文件，以备异常时数据恢复，然后修改内存memtable。
2. memtable使用Skiplist（有序、支持范围查找、增删改不需要调整内部结构）实现保证key有序，定时或按固定大小被刷到磁盘。
3. 定时对持久化文件进行Compaction，减少冗余数据。

memtable落盘写入sst文件，sst文件分层管理Level 0 文件由memtable Flush生成，其他层级由上一 层的sst文件Compaction生成。Level 0 文件内部有序， sst之间无序，key有重叠，Level N 文件内部有序， sst之间有序，key无重叠。Level = 0时，可能有多个文件参与Compaction因为key有重叠，Level > 0时，选择一个文件进行Compaction，在Level+1层中选择与该文件中的key range有重叠的所有文件进行多路归并排序生成新文件写入Level+1层，并删除参与Compaction的文件。

## 一致性协议
CockroachDB最小的存储单元是Range，一个Store中有多个Range，多个Range跨节点部署组成一个Replicate，一个Replicate内的Range采用raft强一致协议，Raft之前有所介绍不在赘述（见[Raft介绍](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/raft/overview.md#raft%E6%A6%82%E8%BF%B0)）。

## 节点状态管理
CockroachDB采用去中心化的节点管理方式使用Gossip协议，关于Gossip协议之前已经介绍过不在这里描述具体细节（见[Gossip介绍](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/overview.md#gossip%E5%8D%8F%E8%AE%AE%E6%A6%82%E8%BF%B0)）。

Goosip的消息传播分为两种模式：
1. 反熵：每个节点周期性地随机选择其他节点，相互交换自己的所有数据来消除两者之间的差异。完全容错，但是网络、CPU等系统资源开销较大。
2. 谣言传播：一个节点有新信息，周期性地联系其他节点向其发送新信息。网络、CPU等系统资源开销相对小一些，但是容错行较差，传播时间相对较长。

之前介绍的RedisCluster在心跳检测中ping、pong、新节点加入中使用的反熵模式，在failover、migrate使用的是谣言传播模式。CockroachDB同样适用两者相结合的方式，在Schema元数据变更时使用谣言传播模式，在Node探活、路由信息等使用反熵模式。

CockroachDB官网根据路由组织方式（后面介绍）给出理论上可以支持1w节点规模，根据之前ReddisCluster的介绍，单条记录大小100B的800节点已经是[RedisGossip](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/ping_pong.md#pingpong)的生产环境上线。CockroachDB传播的数据结构见[CockroachDB Gossip RPC](https://github.com/cockroachdb/cockroach/blob/master/pkg/gossip/gossip.proto)，在单节点的情况下大于100B，故官网的数据有待验证，目前业界已知比较大规模的集群是10个节点4T数据。


## 路由管理
CockroachDB的路由使用连续范围分片，而不是NoSQL常见的一致性hash（猜测主要原因是一致性hash对范围查找不太友好）。CockroachDB采用了二级路由模式（主要是考虑整集群的容量，但是需要三次网络转发），路由信息存储在Range内，默认一个range是64M，一条路由信息是256B，理论上支持 256K * 256K * 64M = 4EB的大小（一级路由支持16T）。

CockroachDB的一级路由通过Gossip传播，二级路由存储在Store内。Range在超过配置大小后会进行分裂，先在本地复制Range，选择Mid Key更改路由信息，异步GC删除连个Range多余的数据。
