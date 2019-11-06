# Replication

命令volume.fix.replication对副本进行修复，对于副本少于规定的会进行补充（多余的不处理），加-n只打印信息不进行执行。

### 流程

1. 向master发起VolumeList请求，获取全量的volume拓扑信息。
2. 根据拓扑信息，梳理vid的分布（ReplicaPlacement > 0）、volume的信息（ReplicaPlacement > 0）、以及所有dn的分布信息（dc、rack）。
3. 扫描所有的volume分布信息，筛选出副本不足的volume及其分布
4. 对所有dn的分布信息进行排序，标准是free的大小（大的在前）。
5. 扫描上述排序的dn分布信息，判断是否满足容量个ReplicaPlacement 对于 副本确实的vid。
6. 如果满足要求，调用VolumeCopy请求给目标节点让其copy源节点的数据。

注意：VolumeCopy是核心逻辑见move的copy部分，不在此赘述源码分析。

### 问题

1. 上述只调用了move中的copy部分，既没有tail部分逻辑这个可能造成数据的丢失。
2. 如果没有tail，也要有只读阶段和机制保证数据完整性，显然上述逻辑没有。