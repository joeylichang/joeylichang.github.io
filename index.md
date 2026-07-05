---
layout: default
title: Joeycli Blog
---

<style>
.profile-actions {
  display: flex;
  justify-content: flex-end;
  margin: 0 0 28px;
}

.github-profile-button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-radius: 4px;
  background: #0d8bf2;
  color: #fff;
  font-weight: 700;
  line-height: 1;
  text-decoration: none;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.25);
}

.github-profile-button:hover,
.github-profile-button:focus {
  background: #0878d8;
  color: #fff;
  text-decoration: none;
}

.github-profile-button svg {
  width: 20px;
  height: 20px;
  fill: currentColor;
  flex: 0 0 auto;
}
</style>

<div class="profile-actions">
  <a class="github-profile-button" href="https://github.com/joey0612" target="_blank" rel="noopener">
    <span>View on GitHub</span>
    <svg viewBox="0 0 16 16" aria-hidden="true">
      <path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82A7.65 7.65 0 0 1 8 3.86c.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"></path>
    </svg>
  </a>
</div>

# Navigation

* **[Overview of NoSQL distributed systems](https://github.com/joey0612/joeylichang.github.io/blob/master/src/nosql_desigin/nosql_distributed_systems_desgin.md)**

* **Distributed protocol**
  * Paxos ([PhxPaxos](https://github.com/joey0612/joeylichang.github.io/blob/master/src/distributed_protocol/paxos/paxos.md))
  * Raft ([braft](https://github.com/joey0612/joeylichang.github.io/blob/master/src/distributed_protocol/raft/overview.md))
  * Gossip ([RedisCluster](https://github.com/joey0612/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/overview.md))
* **RPC**
  * [Summary of the RPC](https://github.com/joey0612/joeylichang.github.io/blob/master/src/rpc/overview.md)
  * [brpc](https://github.com/joey0612/joeylichang.github.io/blob/master/src/rpc/brpc/overview.md)
  * [seastar](https://github.com/joey0612/joeylichang.github.io/blob/master/src/rpc/seastar/seastar.md)
* **Redis**
  * [Common Architectures for Redis-Cluster](https://github.com/joey0612/joeylichang.github.io/blob/master/src/redis/common_architectures.md)
  * [Twemproxy](https://github.com/joey0612/joeylichang.github.io/blob/master/src/redis/twemproxy.md)
  * Twemcache
  * RocksDB
* **SSDB**
  * [source code](https://github.com/joey0612/joeylichang.github.io/blob/master/src/ssdb/souce.md)
  * [LevelDB](https://github.com/joey0612/joeylichang.github.io/blob/master/src/leveldb/overview.md)
* **DynamoDB**
  * [overview](https://github.com/joey0612/joeylichang.github.io/blob/master/src/dynamo/overview.md)
  * [distributed architecture design](https://github.com/joey0612/joeylichang.github.io/blob/master/src/dynamo/desgin.md)
* **CockroachDB**
  * [overview](https://github.com/joey0612/joeylichang.github.io/blob/master/src/cockroachdb/overview.md)
  * [storage layer design](https://github.com/joey0612/joeylichang.github.io/blob/master/src/cockroachdb/desgin_kv.md)
  * [distributed transaction](https://github.com/joey0612/joeylichang.github.io/blob/master/src/cockroachdb/desgin_transaction.md)
* **SeaweedFS**
  * [overview && think in](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/overview.md)
    * data organization
      * [master meta data](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/master/tupo/tupo.md)
      * [volume data organ](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/volume_server/data_type/organization.md)
    * [main process](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/overview.md#main_process)
    * [shell tool](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/overview.md#weed_shell)
  * [optimize design overview](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/opt/opt_design_overview.md)
    * [repair-cluster desgin](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/opt/repair.md)
    * [alloc-cluster && volume-service desgin](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/opt/alloc_read.md)
    * [experience sharing](https://github.com/joey0612/joeylichang.github.io/blob/master/src/seaweed/opt/experience_sharing.md)
* **Min.IO**
  * [overview](https://github.com/joey0612/joeylichang.github.io/blob/master/src/minio/minio_introduce.md)
  * [thinkin](https://github.com/joey0612/joeylichang.github.io/blob/master/src/minio/minio_thinkin.md)
* **Tera(C++ HBase)**
  * [overview](https://github.com/joey0612/joeylichang.github.io/blob/master/src/tera/overview/overview.md)
  * [master introduce](https://github.com/joey0612/joeylichang.github.io/blob/master/src/tera/overview/master_overview.md)
  * [tabletnode introduce](https://github.com/joey0612/joeylichang.github.io/blob/master/src/tera/overview/tablenode_overview.md)
  * [procedure introduce](https://github.com/joey0612/joeylichang.github.io/blob/master/src/tera/overview/procedure_overview.md)
  * [think in](https://github.com/joey0612/joeylichang.github.io/blob/master/src/tera/overview/thinkin.md)

* **Reading notes**
  * [大规模分布式存储系统](https://github.com/joey0612/joeylichang.github.io/blob/master/src/read_node/distributed_system_design/navigation.md)
  * [数据密集型应用系统设计](https://github.com/joey0612/joeylichang.github.io/blob/master/src/read_node/data_intensive_sys_desgin/navigatiom.md)

## Support / Donate

If my articles have helped you, feel free to support me:

- **ETH / USDT (ERC20)**: 0x94b7e791a3dda2999c6789fce6ee0066b078570f
- **BTC**: 1M9ni6JAqbfUfCzo5ftKQNuxLzrkckB5My
- **Others**: Ftqx5D2dTP5wRqS5iZAdy4JB76xEuvjoARNsNxSSeMuP

Thank you for your support! 🙏
