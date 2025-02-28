# op-batcher

op-batcher 的作用就是将 L2Block 提交到 L2Chain 上，用于 op-node 进行进行派生确认 L2SafeBlock 和 L2FinalizedBlock，
核心部分有三个：[submit data](./1_op_batcher_data.md)，SwitchDAType，SendTransaction，[op batcher reset](./2_op_batcher_reset.md)。

SwitchDAType 的作用是根据线上 GasEstimate 调整使用 CallData 还是 BlobData 进行提交 L2Block 数据，触发条件和预估计算比较固定，没有什么特别。

SendTransaction 主要是发送交易给 BatcherIndexAddr，这里值得说的是交易发送的过程是通过队列进行管理的，
队列中 pending 的交易数量是配置的（25），超过这个闸值后面的交易就需要等待前面的完成。如果前面没有超过配置
的闸值，就会立刻返回，但是底层队列中的每个元素是同步的等待，结果返回之后通过 receipyCh 返回给 [ChannelManager](./1_op_batcher_data.md#channelmanager)。
所以，对于上层业务来说交易是并发的（pending 的配置值，目前是 25），但是交易应该是按 FIFO 顺序执行的，因为
目前的配置是一个 block 内只能有一笔这个交易，但是失败了重试之后发送的数据可能是乱序的，所以不能认为 L2Block
是按顺序上链的，[op-node.BatchQueue](../op_node/4_op_node_derivation_data.md#batchqueue) 解析数据的时候会规整数据。