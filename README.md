# Navigation

*  **[Overview of NoSQL distributed systems](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/nosql_desigin/nosql_distributed_systems_desgin.md)**

* **Distributed protocol**
	* Paxos ([PhxPaxos](https://blog.csdn.net/weixin_41713182/article/details/88147487))
	* Raft ([braft](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/raft/overview.md))
	* Gossip ([RedisCluster](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/overview.md))
* **RPC**
	* [Summary of the RPC](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/overview.md)
	* [brpc](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/overview.md)
	* [seastar](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/seastar/seastar.md)
* **Redis**
	* [Common Architectures for Redis-Cluster](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/redis/common_architectures.md)
	* [Twemproxy](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/redis/twemproxy.md)
	* Twemcache
	* RocksDB
* **SSDB**
	* [overview](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/ssdb/overview.md)
	* [source code](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/ssdb/souce.md)
	* [LevelDB](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/leveldb/overview.md)
* **DynamoDB**
	* [overview](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/dynamo/overview.md)
	* [distributed architecture design](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/dynamo/desgin.md)
* **CockroachDB**
	* [overview](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/cockroachdb/overview.md)
	* [storage layer design](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/cockroachdb/desgin_kv.md)
	* [distributed transaction](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/cockroachdb/desgin_transaction.md)
* **SeaweedFS**
	* [overview](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/overview.md)
	* data organization
		* [master meta data](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/master/tupo/tupo.md)
		* [volume data organ](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/volume_server/data_type/organization.md)
	* [main process](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/overview.md#main_process)
	* [shell tool](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/overview.md#weed_shell)
* **tera**
	* [overview](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/tera/overview/overview.md)
	* master introduce
		* master arch
		* master init
		* master state machine
		* master period task
		* tabletnode state machine
		* disaster desgin
		* load balance
		* garbage collection
		* access control
		* quta
	* tabletnode introduce
		* tabletnode arch
		* leveldb opt
	* procedure
		* table procedure
			* create table
			* disable table
			* enable table
			* delete table
			* update table
		* tablet procedure
			* load tablet
			* unload tablet
			* move tablet
			* spilt tablet
			* merge tablet
		* client procedure
			* update(write/delete)
			* read
			* scan
			* compact
		* transaction
			* single row transaction
			* global transaction
	* other
		* monitor
		* observer
	* thinkin
		
* **Reading notes**
	* [大规模分布式存储系统](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/read_node/distributed_system_design/navigation.md)
	* [数据密集型应用系统设计](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/read_node/data_intensive_sys_desgin/navigatiom.md)
