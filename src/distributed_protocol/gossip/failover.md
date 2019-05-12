# RedisCluster Failover

 在介绍Failover只要需要先理解gossip协议中传播的两个版本号的含义，即currentEpoch、configEpoch，RedisCluster的Failover在[官方文档](https://redis.io/topics/cluster-spec)中有详细的介绍在这里只做方案的介绍，RedisCluster Failover分为三种：
1. Failover：Gossip协议选出新主，数据可能有小部分无损。
2. Failover Force：指定节点提升为主，无数据损失，短时无法响应用户请求，业务重试即可。
3. Takeover：强制节点提升为主，数据大概率损失较大，常用于跨地域或者机房故障时跨地域、跨机房强制Failover。

## currentEpoch && configEpoch
版本号在一致性协议中作为信息持续向前更新的常见方式，RedisCluster的currentEpoch 和 configEpoch与Raft的term、Paxos的proposeID设计出发点都是相同的，但是也有一点区别。主要区别是，Raft或者Paxos都是只是针对一个sharding（一主多备份）内的节点，RedisCluster Gossip协议是在全集群内同步拓扑信息所以需要管理全部sharding的信息。

currentEpoch是一个全局的版本号通过Gossip协议传播收敛一致（ping或者pong中的currentEpoch与当前节点比较，更新为较大的currentEpoch）。如果发生主从切换参与选主的节点提升currentEpoch为currentEpoch+1发起投票如果获取了多数的投票，则将configEpoch 更新为currentEpoch。

如果说currentEpoch是一个逻辑版本号，每次集群发生Failover都会提升一次，那么configEpoch就是逻辑时钟的一个tag，他记录了每次Failover时刻的逻辑时钟，每个sharding内新主的出现不会是统一时刻（Gossip通过比较nodeid优先小的进行）所有configEpoch在每个节点内不重复，并且小于等于当前的currentEpoch。configEpoch主要的作用就是控制Failover的是逐个顺序进行的。

_注意：configEpoch的变更严格的说是标识拓扑信息的变化，拓扑信息的变化不仅仅实在Failover，还有Migrate下篇文章介绍_


## Failover 
RedisCluster默认的Failover是自动的，无需外部命令或者模块参与，对业务的影响也是最小的，其他的两种方式都是用来处理一些特殊场景，对业务相对影响较大。其他两种方式都是在默认Failover的基础上做了一定的删减。下面看一下默认的Failvoer流程：

1. Failover发起条件
	1. sharding内slave判断主故障，即Fail状态，必须有slave发起。
	2. Maste有slot（代表Master负责数据）。
	3. slave与master之间复制数据的连接，断开没有超过一定的时间，目的是保证slave数据的新鲜。
2. 投票过程
	1. slave提升自己的currentEpoch为currentEpoch + 1。
	2. 向其他sharding的master发送FAILOVER_AUTH_REQUEST，超时时间 2 * NODE_TIMEOUT。
	3. master投票给该节点的标准是：
		1. slave的currentEpoch必须大于自己的currentEpoch，并记录投票的currentEpoch到lastVoteEpoch。
		2. master的判断故障sharding的旧主为Fail状态。
		3. 一旦同意，将回复FAILOVER_AUTH_ACK，并在2 * NODE_TIMEOUT内不会投给其他node。
	4. slave会摒弃任何currentEpoch小于自己currentEpoch的投票，currentEpoch在这里代表逻辑时钟，表示不是本轮投票。
	5. 获得多数投票的slave获胜，如果超过 2 * NODE_TIMEOUT 终止本轮投票，等 4 * NODE_TIMEOUT之后重新发起投票。
3. 通知集群
	1. slave会将当前的currentEpoch复制给configEpoch。
	2. slave提升为master之后立刻通过Gossip协议向集群内所有节点通知，发生了主从切换并更新每节点内的拓扑信息和currentEpoch（保证全集群的currentEpoch一致且最大）。


上述投票过程需要注意几点：
1. 投票有超时时间除了防止slave死等之外，还给slave提升为master提供足够的时间。
2. 投票具有有效期，保证一旦Failover失败之后，下一轮投票的顺利进行。
3. 通过currentEpoch判断投票轮次。
4. 只有Master参与投票。

如果是一主多备多个slave同时发起投票也回有概率选不出来，所以slave发起投票是有顺序的：
* delay = 500 + random % 500 + SLAVE_RANK * 1000  ms
	1. 500ms，给集群一个反应时间，让其他sharding的master判断旧主为Fail状态。
	2. random增加随机性。
	3. SLAVE_RANK是slave的排序，最大偏移量的排序靠前，优先发起投票，增大了胜算几率，加快数据同步时间。

## Failover Force
Failover Force是指定一个slave提升为master，常用于一些运维场景比如机房切换、机器裁撤等。在上述的投票过程结束之后会与旧主联系，告知其降为slave，降级过程如下：
1. 旧主会停止写请求返回try_again。
2. 旧主将与新主之间的数据gap发送给新主。
3. 发送完毕之后新主通知集群发生了主从切换。


## Failover Takeover
Failover Takeover是强制提升一个slave为master，没有投票过程、没有数据gap同步过程，直接currentEpoch提升通知集群有发生了主从切换。这里有一点需要注意，多个shardings同时进行Failover Takeover会有currentEpoch冲突的问题，虽然RedisCluster Gossip提供了currentEpoch冲突的解决方案，但是那样的收敛时间会非常长，建议还是逐个sharding进行。

Failover Takeover的主要应用与某个地域或者某个机房故障场景，无法完成投票和数据gap协商，需要强制提升为master。