# RedisCluster拓扑传播

## 拓扑数据结构
在探讨RedisCluster实践Gossip之前，应该先明确RedisCluster需要Gossip传播什么样的信息，这个信息不仅可以帮助理解RedisCluster，更是Gossip协议性能分析的基础之一，如果消息很大Gossip协议传播性能必然会受影响，集群的收敛时间肯定慢。RedisCluster传播的数据的结构如下：

```
typedef struct {
    char nodename[CLUSTER_NAMELEN];
    uint32_t ping_sent;			// 向这个节点发送ping的时间点
    uint32_t pong_received; 		// 接收这个节点pong的时间点
    char ip[NET_IP_STR_LEN];
    uint16_t port;
    uint16_t cport;             	/* cluster port last time it was seen */
    uint16_t flags;             	// 节点标志
    uint32_t notused1;
} clusterMsgDataGossip;
```

clusterMsgDataGossip结构体大小是96字节，每个节点内部会为集群内所有的节点维护一个这样的数据结构，例如：一个800节点的集群，每一个节点会维护799个这样的数据结构。

## Ping、Pong
RedisCluster采用的与之前介绍的三种反熵机制（Push、Pull、Push-Pull模式）不同，反熵机制保证经过1-3轮交互之后两个节点的信息相同，RedisCluster使用的是谣言传播机制传播clusterMsgDataGossip数据，即当一个节点有来新信息后，该节点变成 活跃状态，并周期性地联系其他节点向其发送新信息。RedisCluster采用的与之前介绍的三种反熵机制每次之传播10%的节点数据（10%的节点数少于3个传播3个）。RedisCluster内消息的发送与接收通过ping、pong包完成，具体逻辑如下：
1. 当前节点在NODE_TIMEOUT/2 之内向所有的节点发送一遍ping包，ping包中包含10%节点的clusterMsgDataGossip数据，以及当前节点的信息描述clusterMsg（数据相对clusterMsgDataGossip更全面一些包括key映射的slot信息等）。
2. 其他节点收到ping之后即可回复pong包，pong包的内容与ping相同即clusterMsg + 10%节点的clusterMsgDataGossip。
3. 如果在NODE_TIMEOUT/2 没有收到某个节点的pong回复标记该节点为PFail，会在NODE_TIMEOUT/2之后立刻重建连接（防止因为连接问题导致未回复）再向其发送ping消息，如果在 TIME_OUT/2 内仍未回复pong，则标记该节点为Fail。

_注意：集群中每个节点都会执行上面的逻辑进行探测（Gossip协议的理论）_

接下来看一下集群在一个周期内（NODE_TIMEOUT）的Gossip的发包量情况，假设集群包括800个节点，NODE_TIMEOUT为15s：
* 一个节点在NODE_TIMEOUT/2 = 7.5s内需要发出799个ping包、接收799个pong包，每秒发送、接收106.5个ping包。
* 全集群800个节点，集群每秒发送、接收85.2k个数据包。
* 一个ping包包括一个clusterMsg和79个clusterMsgDataGossip，为方便起见clusterMsg和clusterMsgDataGossip都按100字节计算（clusterMsg大小在96 ~ 130左右字节之间，clusterMsgDataGossip比较固定96字节），一个ping包大小为8K。

Reds是基于epoll + 回调的单线程处理模型，处理的连接数 和 数据量大小都是有限的，一个redis节点每秒处理106.5个8K的数据包（注意这里不包括用户的请求，redis单点可以处点每秒可以处理10w string类型用户请求已经成为共识）压力还是不小的，这也正是Gossip协议被诟病的点。

给出一组生产环境的数据感受一下gossip传播消息的收敛时间，上述场景800节点中bj部署400节点（同地域节点ping耗时1ms以内），nj、gz个200节点（bj到nj ping耗时20ms，bj到gz ping耗时40ms），集群发生主从切换之后（下面文章介绍）通过RedisCluster实现的Gossip协议完成集群拓扑收敛一致的时间在60s - 75s，对业务影响还是不可小觑的。

Redis官方给出的建议的是不超过1000个节点，在上述的跨地域部署中800个节点对业务影响已经不少了，根据经验上述部署方案如果降到640个节点（bj 320个节点，nj、gz 160个节点）拓扑的收敛时间可以降到10s左右，可以看出集群规模与Gossip协议传播时指数级增长关系，如果下降到480节点收敛时间可以在5s以内对业务也想就会小很多。


## 节点加入
RedisCluster加入新的节点通过两种途径：
1. 发送Cluster Meet（指定要加入节点的ip port）命令给集群中任意一个节点。
2. 通过ping、pong传播，将新节点传播给其他节点，其他节点与新节点handshake。
