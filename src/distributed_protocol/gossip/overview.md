# Gossip协议概述

Gossip协议即谣言协议或者八卦协议，具有去中心化、最终一致、分布式容错等特点。在很多开源项目和国外的分布式系统中有很多应用，例如：Amazon S3、Amzon Dynamo、Cassandra、Redis、CockroachDB等等，其中相比较Google传统的GFS、BigTable、Spanner架构体系中常见的Paxos强一致协议，Amazon更青睐最终一致的Gossip协议。

本文首先简单介绍Gossip原理，然后结合RedisCluster讲解Gossip协议在其中的应用，其中在适合的章节还会分享一下Gossip协议在生产环境遇到的挑战。

## Gossip协议
### 理论基础
在一个有界网络中，每个节点都随机地与其他节点通信，经过一番杂 乱无章的通信，最终所有节点的状态都会达成一致。每个节点可能知道所有其他节 点，也可能仅知道几个邻节点，只要这些节点可以通过网络连通，最终他们的状态 都是一致的，类似于疫情传播。

### 去中心化 && 分布式容错
gossip协议不需要节点知道所有其他节点，节点之间完全对等，不需要任何中心节点。常见的中心化解决方案需要一个类似“master小集群”的解决方案作出裁定，比如：节点是否故障等，如果中心节点出了问题整个集群会面临服务降级甚至不可服务的境地。对于去中心化的解决方案，任何一个点故障（严格说少于半数节点故障）集群都能做出裁定，都可以保证集群正常服务，甚至在网络分区的情况下也是可以保证数据一致性的。所以Gossip协议是分布式容错的，他的容错能力与集群中节点数量成正比，节点越多容错能力越强，完全不依赖外部模块（“master小集群”一般3或5台）。


### 最终一致
根据上面的描述可以知道，所有节点状态收敛一致需要一段时间，这也是gossip被称作最终一致性协议的原因，这段时间的长短与集群中节点的数量成指数级增长，这也是gossip协议最受诟病的地方，但也不是没有没方案解决。Amazon S3单集群在全球的部数量达到10+w台物理机通过gossip协议管理节点就是例证，在[Redis常见集群架构](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/redis/common_architectures.md#%E5%A4%A7%E8%A7%84%E6%A8%A1%E7%9A%84%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88)中会尝试提出一种解决方案。

在Gossip协议中节点越多，分布式容错能力越强，即可以容忍更多的节点同时故障（少于半数），故障的恢复时间越长。

### 传播方式
* Push模式
	1. 节点A将全部数据推送到节点B，B更新比节点A陈旧的数据，Push完成后，A数据保持不变，B被A感染;
	2. 1个通信周期
* Pull模式
	1. 节点A发送摘要数据(即数据版本)到节点B，B将本地比A新的数据发送到A，A更新 本地数据;
	2. Pull完成后，B数据保持不变，A被B感染;
	3. 2个通信周期
* Push-Pull模式
	1. A将摘要数据发送到B，B将本地更新的数据发送到A，同时，B将本地比A老的数据生成摘要也发送给A，A更新本地数据，同时将A中更新的数据发送给B，B更新本地数据;
	2. Push-Pull完成后，A/B数据都有更新，交叉感染;
	3. 3个通信周期
* Push-Pull模式的收敛速度最快， Pull模式次之，Push模式最慢。

## RedisCluster
RedisCluster是Redis开源项目提供的官方集群解方案，目前在一些国内的知名互联网公司中都有应用。RedisCluster通过Gossip协议实现了集群拓扑信息传播与管理、自动failover、透明数据迁移等。本文通过以下几部分介绍Gossip协议在RedisCluster中的实践：
1. [RedisCluster tupo spread](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/ping_pong.md)
2. [RedisCluster failover](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/failover.md)
3. [RedisCluster data migrate](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/migrate.md)
