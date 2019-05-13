# RedisCluster data migrate

## key 组织方式
RedisCluster的数据迁移是以slot粒度进行的，需要先介绍一下key的组织方式。RedisCluster的key通过slot进行组织，key经过crc校验和计算之后对16384（16KB）取摸作为key的归属slot，集群中每个sharding负责若干个slot。

Redis节点中除了通过字典维护一个全局的k-v表，为了方便数据的迁移会为每个当前节点负责的slot再维护一个数据结构用于存储每个slot有哪些key。一个key在redis节点中会被存储两份，造成了内存的一定浪费，例如：有的业务存储的sizeof(key) 大于 sizeof(value)，在最早的版本用的是也是字典后来优化为RadixTree（字典树的优化）来节省空间。

## 透明迁移

### 搬迁过程
Rediiscluster为了保证数据迁移对用户透明对slot增加了Migrating、Importing状态，并且添加了Asking、Moving语义和命令。具体流程如下：

1. 对需要迁移的slot在目标节点设置为importing状态。
2. 对需要迁移的slot在源节点设置为migrating状态。
3. 使用migrate命令在源dump一批key并进行压缩，再写入目标节点，整个过程是原子的，只有migrate保证成功之后才会删除这批key在源节点的数据。
4. 如果在迁移期间有用户请求，处理逻辑如下：
	1. 如果访问key在源节点，直接在源节点操作。
	2. 如果访问的key不在源节点且slot处于migrating状态，返回moving语义（包括目标节点的ip、port）。
	3. 客户端收到moving语义之后向目标节点发送asking命令，目标节点会检查slot是否是importing状态，是的话返回同意，客户端可以直接向目标节点操作（新数据或者已经搬迁的都在目标处理），asking之后只能有一次操作，之后再有请求还需先访问源节点，重复上述步骤。
	4. 如果访问的key正在执行migrate命令，返回try_again。
5. 迁移完成之后擦除目标、源节点slot的migrating、importing状态。

### 通知集群
RedisCluster的拓扑信息都维护在clusterMsg中，概括起来主要是主从关系、slot归属。这两个发生了变化就需要RedisCluster Gossip在集群内传播，data migrate是的slot归属节点发生了变化，所以需要提升currentEpoch并向全集通广播传播。

### 风险
RedisCluster的数据迁移主要目标是对用户透明，这使得他也暴露了一些其他问题：

1. 数据搬迁过程较慢，因为一次只批量搬迁小部分key，为了保证对用户影响尽量小（减少上述的try_agagin）每次搬迁的key在几十到几百并不是很多。

2. 如果搬迁结束之后通知集群时也发生了主从切换（看起来概率很小但还是会发生）currentEpoch就会有冲突的情况（因为migrate提升currentEpoch时没有想Failover那样进行协商是自己强制提升的，所以一定会冲突），集群的收敛时间就会变长。

3. 数据迁移分批迁移成功之后就会立即删除源上的数据，如果此时目标节点故障就会有丢失数据的风险，即使目标节点有备份（RedisCluster的主备之前是异步同步），也是有可能丢失一部分数据的。
