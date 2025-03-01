# op batcher reset

op-batcher 的 reset 情况比较单一，只需要考虑 L2 Reorg，因为 L1 的 Reorg 不需要考虑只会影响提交上去的 L2Block 
被回滚了，业务层并没有看到这部分的处理，猜测应该是 txmgr 处理了，他会等到交易上链的 block 被 finalize 之后才返回
成功（这对于 fast finality 来说正常情况下会是很快的）。

L2 Reorg 的情况只有一种情况下，即 [ChannelManager.AddL2Block](./1_op_batcher_data.md#channelmanager) 
的时候会校验最后一次插入的 block 和要插入的 block 的连续性，如果不连续返回 `ErrReorg` （在主循环中其他的错误将被忽略），
此时主循环会等待现有的 数据全部提交之后，向 op-node 发送 rpc 请求询问目前的 L2SafeBlock，一次作为基地进行
reset [op batcher submit data](./1_op_batcher_data.md)。 询问 op-node 是最可靠且最准确的，因为说明这个结果被派
生验证过了没有问题，但是可能没有同步给 op-geth。