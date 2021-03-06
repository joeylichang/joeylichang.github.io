## Alloc-Cluster

### 背景

<img src="../../../images/seaweed_arch_opt.png" alt="seaweed_arch_opt" style="zoom:50%;" />

如上图所示，Alloc-Cluster 在整个系统架构中负责 Client 申请 <vid,fid> 的请求（既 Assgin 请求），将该部分功能从 Master 剥离的主要目的是分摊 Master 压力，可以支撑更高的写入性能（灌库、滚库等业务需求）和更大规模的集群，随着 Alloc 模块的部署集群的写入能力成线性增长（单 Master 存储性能上限）。

如果没有 Alloc-Cluster 随着集群规模的扩展（千台以上），系统的写入能力不与机器规模成正比，此时 Master 承接的 Assgin 请求成为瓶颈，尤其是在批量灌库的场景中。



### 详细设计

Alloc 模块的详细设计中有两个难点：

1. 申请的  fid 必须是全局唯一（重复可能会造成用户数据覆盖），且局部递增（可能一次会申请多个 fid）。
2. Alloc 需要实时掌握全局的拓扑信息，用于 vid 的分配。



#### Fid 设计

首先，需要明确 Fid 全局唯一的含义，表面含义 Fid 是用户数据的唯一标识，如果只是这样的话对用户数据的某一部分（比如 key，不用全部数据可以加快签名速度）进行签名（Base64等）生成唯一的识别 id 即可满足。同一份数据二次写入时可能（注意是可能，因为不一定会分配到相同的 vid）会进行覆盖，这是不符合对象存储语义的（以S3作为标准），对象存储的共识是一个追加写系统（既，相同的用户数据会被写两次，会用不同的 fid 进行表示，没有覆盖语义，只有删除 + 重写）。

故， Fid 的隐藏含义是对象存储系统内的数据唯一表示，而不单纯是用户数据的标识，既对象存储系统不区分存储的数据是否相同，更不考虑覆盖的问题，在系统内部都认为是不同的，覆盖是用户通过删除旧数据+写新数据完成。

上述论述都是基于 Amazon S3 语义为基础，目的是阐述 Fid 全局唯一且局部（局部主要是满足批量申请多个 fid的场景）递增的必要性，也是设计和实现阶段无法避免的问题。



###### 有状态模块

Alloc-Cluster 正是 Fid 全局唯一且局部递增 的特性，使其成为了有状态服务，需要 Master 维护全局最大的 Fid 和 分配（分配个不同的 Alloc 节点）。从 Fid 分配的维度上看 Matser 和 Alloc-Cluster 更像是 Id-Alloc 服务的缩影。针对 Id-Alloc服务，美团的技术团队发表的博文《[美团分布式ID生成服务开源](https://tech.meituan.com/2019/03/07/open-source-project-leaf.html)》有将详细的论述，并且针对全局唯一且递增的场景提供了一套可靠的分布式解决方案，不在此过多赘述。seaweed 场景没有那么严苛，只需要全局唯一且局部递增即可，具体方案如下：

Alloc 每次向 Master 申请一批 id 区间（区间大小默认100w），Alloc 模块快使用完这批 id 之后（默认使用 90w）异步向 Master 重新申请一批新的 id 区间，待当前id区间使用完毕后切换新的id区间。Master 每次分配 fid 之后需要通过raft向其他 Master 同步，并分别持久化（使用raft 统统不 和 持久化，社区版seaweed不支持）。Alloc 模块没使用完一定数量的 id （默认 10w）会本次持久化一次。

Alloc 重启会先从本低持久化的 id开始 加 10w 作为起始值进行分配，做到了不重复并减少了 id 的浪费。如果 Alloc 机器故障无法恢复，其上分配的 100w 个 id 中未使用的直接丢弃。同一个 Alloc 两次申请的 id 区间大概率是不连续的，因为其他 Alloc 服务可以申请。

从用户侧看申请的 fid 一定是全局唯一的，因为 Alloc 模块分配给用户的 fid 是从 Master 获取的，Matser 保证了分配的递增且连续。用户同一个请求（一次申请多个 fid）或者同一个链接申请的 fid 一定是全局递增的。

**注意：有一种临界状态，既一次申请多个 fid，正好当前id区间使用完毕，此时分配的多个fid可能是不连续的，后面介绍该解决方案**



#### Topo 更新设计

Alloc 模块的功能主要有两个一个是分配 fid，另外一个是分配 vid，前面介绍了 fid 分配的设计方案。下面看一下 vid 的分配方案。

Alloc 分配 vid 必须要掌握全局的拓扑信息，才能进行最优的 vid 分配。Alloc 获取全局拓扑的信息的方式比较多，例如 pull、push、push-pull 等。pull 的方式是从 Master 拉取拓扑信息更新本地，又可以分位全量更新和增量更新，全量更新既每次从Master拉取全量拓扑信息更新本地，增量更新既每次只同步拓扑的变化量。push 方式将 Alloc 本地的拓扑信息发送给 Master，Master 进行比对将 diff 部分返回给 Alloc。pull-push 方式既将本地拓扑更新发送给 Master，Master 返回 Alloc 有哪些更新， Alloc 将关注的 diff 再拉回本地进行更新。

Alloc 模块使用的是 pull 方式的全量更新模式而未使用其他方式，主要考虑以下几点：

1. Push、push-pull 逻辑过于复杂，并且需要 Master 维护 Alloc 的信息，设计不够简洁，同时对 Master 改动较大，不符合优化之初设计的原则（减少对源码的侵入，方便社区优秀 feature 的rebase）。
2. 增量更新需要 Master 维护增量数据，维护起来也归于复杂。
3. pull方式的另一种优化，是 Alloc 与 Master 通信拓扑信息的版本号，当有版本号变更时再拉取全量或者增量数据。因为Alloc 拉取的拓扑信息中除了 volume 分布、volumeServer 等信息，还包括节点的统计信息（qps、流入流出带宽、容量等信息）这些都是 Alloc 分配vid的依据，这些信息变化较为频繁。
4. 经过测试单机 Alloc 在秒级更新拓扑的情况新， Assgin 请求极限性能能够达到 8k，千台规模的集群部署几十台 Alloc 即可（按4k qps计算），Master 承接几十到上百qps的拓扑拉取请求的性能是没有任何问题的。



### 实现

#### IdAlloc

全局维护一个 AllocQueue 数组，数组大小与物理核数相等（发挥多核的并行优势），每个Alloc对象包含  IdPool 和 Topo 两个对象，既一个核心绑定一个 topo 和 一个 id pooll。

每个 IdPool 单独向 Master 发送请求申请 id 区间，并未各自的持久化目录（默认10w持久化一次）。每个 IdPool 内部维护维护两个 Interval 内存结构记录申请的id区间，形成双buffer切换机制。当id使用量超过 90%，则通过 chan 通知 IdPool 启动一个协程去 Master 拉取新的id区间，然后存放到备份 Interval 中，当前 Interval 使用完则进行切换。



#### topo 更新

主协程负责定期拉取远程的拓扑信息，与本地的拓扑信息进行比对，如果拓扑没有变更，且统计信息变化低于一定比例（默认10%），则不进行更新，否则通过管道通知，每个核心上的 Alloc进行进行更新。



#### vid 选取

Alloc 模块掌握了全局的拓扑信息以及volume 和 节点的统计信息，基于此进行 volume 的选择。初期只考虑了 volume 容量，类似 repair 中的策略满足 collection 和 rpl 要求的 volume 作为备选集，剩余容量越多的volume被选中的概率越大。



### 问题

###### volume 只读阶段

repair 设计中加入值 volume 的只读阶段（与社区版中的只读机制是两套，详情可见 repair 部分介绍），alloc 接收到只读有最长 1s 的延时，期间可能有数据的写入。repair 的 copy 使用的是 copy + tail 方式，即使有写入也无妨，只要不是一直有写入即可（最多1s），都可以保证数据不丢失。



###### 批量申请fid

如前所述，在当前 id 区间临近用完是，如果申请的fid 个数大于剩余的id，可能会不连续或者不够用。再次针对 Assgin 批量申请的接口语义进行了修改，分三种情况可供业务选择：

1. 业务可以介绍两段连续，但是两段间不连续的 fid（当前id区间剩余 + 新id区间开始的部分）。
2. 业务可以接受，返回少于指定申请的fid个数（直将当前id区间剩余的id分配给他）。
3. 业务不接受上述情况，进行重试，因为分配的 id区间较大，且 alloc 模块部署不止一台，重试仍然出现的概率不大。

根据业务使用场景来看，使用批量申请的情况并不多（只有大文件拆分时会使用），通过监控来看这种情况发生并不多见。



### VolumeService

VolumeService 供业务查询 vid 的分布，从功能上看是 Alloc 模块的子集，只有 Topo 部分的逻辑并且更简单，因为不需要计算统计信息的变量来决定是否更新拓扑信息，只需要关注dn、volume、collection 的变更即可。 