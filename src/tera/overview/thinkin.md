# Think In

### BigTable Compare

##### Chubby

1. Chubby 介绍
   1. Chubby 提供了一个名字空间，里面包括了目录 和 小文件（与 ZK 区别）
   2. 每个目录或者文件可以当成一个锁，读写文件的操作都是原子的。
   3. Chubby 客户端与 Chubby 通过租期保持 Session，当一个 Session 失效时，它拥有的锁和打开的文件句柄也都失效了。
   4. 当 Chubby 内部的文件或者目录有变更时，客户端会受到通知。
2. 在 BigTable 中的使用
   1. Master 选主。
   2. 存储 BigTable 数据的自引导指令的位置（类似 MetaTablet 的位置）。
   3. 查找 Tablet 服务器，以及在 Tablet 服务器失效时进行善后。
   4. 存储 BigTable 的模式信息（与 Tera 区别）。
   5. 以及存储访问控制列表。（与 Tera 区别）
3. VS. Tera
   1. ZK 不支持文件。
   2. Table Schema 信息 以及 ACL 存储在 MetaTable中。



##### MetaData

![bigtable_metadata_argan](../../../images/bigtable_metadata_argan.png)

1. MetaData 三层索引
   1. 在 Chubby 中的一个 File 保存着 Root Tablet 的位置。
   2. Root Tablet（唯一且永不分裂） 保存着 MetaData Table 所有 Tablet 的位置。
   3. MetaData Table 中保存着其他所有 Table 的 Tablet 的位置。
2. VS. Tera
   1. Tera 的 MetaTablet = Root Tablet +  MetaDataTableTablets，既两层索引。
   2. Bigtable 
      1. 容量更大，METADATA 的每一行都存储了大约 1KB 的内存数据，128MB 的 METADATA Tablet 中，采用这种三层结构的存储模式，可以标识 2^34 个 Tablet 的地址（RootTablet * MetaDataTablet = 2^17 * 2^17）。
      2. 如果客户端的缓存失效，网络IO 次数最多达6次（3次失效 + 3 次请求）。
   3. Tera
      1. 同样大条记录大约 1KB 的内存数据（实际没有这么大），128MB 的 MetaTablet 能够标识 2^17 个 Tablet 的地址（同样情况下容量介绍一半）。
      2. 如果客户端的缓存失效，网络IO 次数最多达4次（2次失效 + 2 次请求）。



##### Spilt Tablet

1. Tera：Master 发起，控制全流程。
2. BigTable：TabletNode 发起，好处是更及时，甚至很多用于判断是否分裂的信息不需要再汇报给 Master 较少网络带宽。
   1. 问题：TableNode 完成分裂之后如果通知 Master 失败（网络、Master、TabletNode 故障），Master 在分配 Tablet 加载时，TableNode（不一定是之前的 TabletNode）会发现因为分裂导致加载不起来（可能是 GFS 层有变化），此时 TableNode 会上报给 Master 与 RootTablet 进行对比，完成两个 Tablet 的加载。



##### Cache

Bigtable 没有 Persistent（SSD 盘）的缓存。



##### WAL

Tera

1. 一个 Tablet 一个 wal 日志，一个 TabletNode 对应多个日志。

Bigtable

1. 一个 TabletNode 对应一个 wal 日志。
   1. 优点是减少了对 GFS 的 seek 并行度。
   2. 缺点是 Tablet 恢复会很慢。
      1. 为此对日志进行排序（table，row name，log sequence number），这样用一个 Tablet 的日志连续排在一起。
      2. 对日志进行 64MB 进行分段，每次恢复的时候由 Master 协调其他节点分段进行排序（一般 wal 日志不会过大，除非批量删除等极少数场景）。



##### 总结

1. Chubby vs. ZK
2. MetaTablet 三级索引 vs. 二级索引
3. SpiltTablet TableNode 发起 vs. Master 发起。
4. 内存耳机缓存 vs. 内存 + SSD 三级缓存。
5. WAL TabletNode 共享（排毒） vs. Tablet 共享。



### Disaster



### Question



### Q&A



1. master故障元数据恢复

2. 全局事务的设计 不够完整 

3. LB策略

4. HDFS

5. kv 表格 CPU利用率

6. 负载均衡 概率，容量搬迁总是选举，并且较为频繁

7. hdfs 小文件

8. TabletNode 没有权限验证

   
