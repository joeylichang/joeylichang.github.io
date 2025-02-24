# op-node derivation data

## L1Traversal

## DataSourceFactory

## L1Retrieval

## FrameQueue

## ChannelBank

## BatchQueue

## FetchingAttributesBuilder

FetchingAttributesBuilder 主要作用是根据 L2ParnetBlock 和 L1Block（epoch） 创建 PayloadAttributes 模板。
创建的 PayloadAttributes 模板默认情况下 `NoTxPool=true` 标识没有定序器打包的交易，sequencer 或者 verifier(FullNode) 模式需要重新设置 `NoTxPool=false` 以便打包交易进区块。

### PreparePayloadAttributes

1. l2.SystemConfigByL2Hash 获取参数中 L2Block 对应的 SystemConfig，代表当前的情况。
   1. 后续要么直接打包进下一个区块，要么收到 L1 的 UpdateSystemConfigTX 对其进行更改并打包进下一个区块。
2. l2Parent.L1Origin.Number != epoch.Number， L1 起始区块发生了变化，那么新的 L2Block 就是当前 epoch 的第一个区块。
   1. 需要从 L1 起始区块获取所有交易收据，以便扫描 deposits txns。
   2. l1.FetchReceipts => receipts, L1BlockInfo
   3. DeriveDeposits(receipts) => deposits
   4. seqNumber = 0
3. l2Parent.L1Origin.Number == epoch.Number
   1. l1.InfoByHash => L1BlockInfo
   2. depositTxs = nil
   3. seqNumber = l2Parent.SequenceNumber + 1
4. 计算 BaseFee
   1. Snow-优先/Fermat-其次/默认
5. `nextL2Time := l2Parent.Time + ba.rollupCfg.BlockTime， if nextL2Time < l1Info.Time() return error` 
   1. 计算下一个 NextL2Boock 的 BlockTime，如小于 L1 的时间就会返回 error。
   2. **L1 一定是 one by one 的处理，所以如果 L2 的出块时间大于 L1 的出块的时间是不行的，L2 会停。**
6. 如果是 Ecotone 和 Fjord 升级的那个 Block，需要增加其对应的 upgradeTxs。
7. 组装 Txns
   1. L1InfoDepositBytes： deposit 类型的 system config tx，每一个 L2Block 都有的，其他的只有 epoch 的第一个有。
   2. depositTxs：用户的质押交易。
   3. upgradeTxs：系统升级的交易，一半是硬分叉。
8. 处理升级
   1. 如果是在 Canyon 之后，需要 withdrawals 交易不能是 nil。
   2. 如果是在 Ecotone 之后，需要增加 parentBeaconRoot。
9. 组装 L2Block 并返回。

## AttributesQueue
