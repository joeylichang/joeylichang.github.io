# Seaweed

## 整体架构

<img src="../../images/seaweed_arch.png" alt="seaweed_arch" style="zoom:50%;" />

seaweed的整体如图所示，主要分为master和volume_server（不包含filer，后续介绍filer和ec部分）两部分。master使用raft协议管理元数据，volume_server负责数据存储。

##### master

seaweed支持按DataCenter、Rack、DataNode的物理单元管理存储单元Volume（存储数据的最小单元，默认30G），并且支持collect管理volume，collection是一个逻辑概念按client需要对volume进行逻辑划分。

本系列文章针对master主要介绍以下几部分：

1. [拓扑数据管理](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/master/tupo/tupo.md)
2. [定时任务](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/master/collect_full_and_garbage.md)
3. [master接口梳理](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/master/master_interface.md)



##### volume_server

volume_server一个进程对应一个Store，一个Store对应多个DiskLoction（对应一个磁盘，更准确是对应一个目录），一个DiskLoction对应多个Volume，一个Volume对应多个Needle，一个Needle对应一份用户的数据。一个Volume包含两个文件，既数据文件和索引文件。

除了存储数据的组织外，volume还支持客户端的读写、集群内部命令，例如mount、unmout、copy、tail等。

本系列文章针对volume_server主要介绍一下几部分：

1. [存储数据结构介绍](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/volume_server/data_type/organization.md)
2. [volume_server接口梳理](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/volume_server/volume_interface.md)



##### main_process

存储系统最近本的操作是读写删除操作，seaweed的写和删除是强一致，既删除主节点数据之后向从节点同步，所有从节点的删除都成功之后才算成功，否则更新失败。

seaweed的读写都需要与master交互，写操作时向master申请fid，fid有三部分组成：volumeId + 全局递增序列号 + cookie，申请fid的结果会返回vid所在的主节点的地址，client向主节点发起写请求。读请求会像master查询fid对应的volume的分布，client收到一个volume_server的列表。

除了读写操作seaweed的心跳汇报也是值得重点介绍的，seaweed分配fid的全局递增id是不落盘的，这样一旦三个master都重启（当然概率会更小）必须保证master收到了所有volume_server的心跳，因为心跳会汇报其最大的fid，这样才能不正后续不重复。再或者用户在申请fid之后串改了中间递增id会使得fid有一定概率重复。

volume中没有使用任何存储引擎，所有的写都是追加写，删除是标记删除，如果删除的较多会造成空洞造成空间浪费，seaweed的解决方案是compact。

本系列针对seaweed主要的流程做一下几部分的介绍：

1. [assign(alloc fid)](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/main_process/assign.md)
2. [read/write/delete](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/main_process/write_read_del.md)
3. [compact](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/main_process/compact.md)
4. [heartbeat](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/main_process/heart_beat.md)



##### weed_shell

以上介绍的是seaweed的核心模块逻辑，但是这个系统在一些容错场景下并不是完美的，作者提供了一些外围的工具进行补偿，命令很多选了一下几个比较重要（影响数据可靠性、可用性的场景或者说运维使用较多的命令）进行一下介绍：

1. [balance](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/weed_shell/balance.md)
2. [replication fix](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/weed_shell/fix_replication.md)
3. [move](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/weed_shell/move.md)



## 思考

##### 



