# op-node derivation data

## Overview

本部分主要是介绍 op-node 解析 L1Block 里面的 L2Block 数据（BatcherAddr上的 BlobData），下面这些数据的关系多数是一个嵌套关系。
最终，**NextAttributes 会返回全局（EngineController）的 pendingSafeHead 之后的下一个 L2Block 用于派生**。其中最核心的是
BatchQueue 负责解析 L2Block 的 SpanBatchData 以及其对应的 L1Origin，最终组装成 L2Block，这里面要做一系列的校验和容错，比如 
L2SpanData 的荣誉，乱序等。

**注意：Derivation Data 部分使用的 L1 客户端是 [confDepth](./1_op_node_clients.md#confdepth)，距离最新块 15 个Block。**

## L1Traversal

L1Traversal 是对当前正在解析的 L1Block 的一层封装，内部维护两个重要的成员变量：
* `block`：记录当前正在解析的 Block。
* `sysCfg`：记录到当前 Block 为止 system config 信息，用于生成 PayLoadAttribute。

### AdvanceL1Block

数据的入口，在每次派生之后（[Step](./5_op_node_derivation_pipeline.md#enginequeuestep)） 会调用它。
内部核心逻辑就是获取 `block + 1` 的数据（BlockInfo 和 receipts）然后更新 `sysCfg`（receipts 的 log）。

有一点比较重要：**`block` 与 `block + 1` hash 会进行校验，不通过说明 L1 reorg 会返回 ErrRest，主循环会调用 Rest**。

### NextL1Block

数据的出口，返回当前的 `block`。

### Reset

参数会传入一个 L1BaseBlock 设置给 `block`，这个 L1BaseBlock 是 L2SafeBlock.L1OriginBlock。
参数还会有一个配套的 SystemConfig 会设置给 `sysCfg`。

## DataSourceFactory

DataSourceFactory 是一个解析当前 Block 上面的 BatcherIndexAddr 地址的数据（CallData 或者 BlobData）。

DataSourceFactory 只提供了 `OpenData` 方法，需要传入 L1Traversal 返回来的 Block 和 配置的 BatcherIndexAddr。
会返回 DataIter 接口，在 ecotone 升级之后，使用的是 BlobDataSource(DataIter 的一个实现)，但是他也会兼容 
CallData 数据，即 CallData 数据也会放入 BlobData 里面。

BlobDataSource.Next 是获取数据的方法也是本部分的重点，返回的是一个 TX（BatcherIndexAddr作为接收者） 中的一个 BlobData（op-bathcer 那边一个交易尽量是 6 个）。

## L1Retrieval

L1Retrieval 是针对 L1Traversal 和 DataSourceFactory 的组合封装，其 NextData 方法就是返回一个 BlobData。
NextData 内部通过 L1Traversal 返回 Block，作为参数传递给 DataSourceFactory ，返回 DataIter 即可用于返回 BlobData。

## FrameQueue

FrameQueue 是在 L1Retrieval 的基础上封装一层，数据的出口是 NextFrame 返回 Frame。
一个 BlobData 对应多个 Frame，所以内部有一个数组存储形成了 Queue。

## ChannelBank

ChannelBank 是对 Channel 的封装，内部会维护一个 ChannelQueue。一个 Channel 内部是有多个 Frame，如果 Frame.IsLast == true 表明是最后一个 Frame，如果一个 Channel
内部全部的 Frame 都全了，则 Channel 进入 Ready 状态，里面的数据就可以读取了，否则不可以。所以一个 Channel 对应完整的一组 Frame。

### Channel

数据的出口是 Reader，在 ChannelBank 调用 Reader 之前会先调用 Channel.IsReady，如果 **ready 才会读 Channel 内全部的 Frame 数据**，IsReady 的标准：
1. Channel 没有 close。
2. Channel 的 inputs 在 [0, endFrameNumber] 之间的数据都齐了， endFrameNumber.IsLastSpan == true.

数据的入口是 AddFrame 将 Frame 添加到 inputs ，并更新相关的元数据（endFrameNumber，close，highestFrameNumber），
当 IsLastSpan = true 时候，会更新 endFrameNumber 和 close，可能有乱序，即 IsLastSpan 的 Frame 会先到，中间的 Frame 后到，这种场景也会被成功添加。

### NextData

1. 数据出口：Read
   1. 判断 channelQueue 的第一个 channel 是否过期，L1Block 跨度 1200 个 —— FIFO，理论上他是需要先淘汰的。
   2. 遍历 channelQueue 开始入数据，第一个读到直接返回：
      1. tryReadChannelAtIndex 从 0 开始逐个便利 channel 进行数据。
      2. 先判断 channel 是否过期，L1Block 跨度 1200 个。
      3. 如果管道是 IsReady，Channel 内全部的 Frame 数据都齐全了。
      4. 然后返回全部 Frame 数据。
2. 数据入口：IngestFrame(FrameQueue.NextFrame)
   1. 判断 Frame 对应的 channel 是否存在，不存在就创建。
   2. 然后 AddFrame。

**注意：Channel 的过期度量单位都是 L1Block 的个数，并不是 L2Block。**

## ChannelInReader

ChannelInReader 主要是将 Frame 转换成 Batch 数据，这里就只看 SpanBatch（目前都是）。
NextBatch 就是把一个 Channel 内全部的数据都拼装并解压缩成一个 SpanBatch。

## BatchQueue

BatchQueue 从 ChannelInReader 中获取 SpanBatch 并拆解为 SingularBatch，然后根据参数 L2ParentBlock 获取到其对应的后面的 L2Block。

### NextBatch

1. nextSpan 是一个数组，内部记录了一个管道内全部的 SingularBatch（与 L2Block 一对一）。
   1. **如果内部有数据，且时间戳 == L2ParentBlcok.Time + 1s，则返回该数据**。
   2. 如果返回的 SingularBatch 是最后一个，返回 IsLastSpan。
2. originBehind 表示当前正在解析的 L1Block（来自[L1Traversal](#l1traversal)）是否落后于 L2ParentBlock.L1Origin.Number。
   1. l1Blocks 记录的是用于匹配 L2Block 的 L1Block，同样来自 L1Traversal 在迭代过程中积累下来的。
   2. 所以 l1Blocks 是单调递增的。
   3. 清理掉小于 L2ParentBlock.L1Origin.Number 的 block 在 l1Blocks 其中的。
   4. 但是等于的需要保留，因为一个 L1Block 可以对应多个 L2Block。
   5. 如果 originBehind == false，表示当前正在解析的 Block 没有落后于 L2ParentBlock.L1Origin
      1. l1Blocks 尾部追加当前正在解析的 L1Block。
   6. 如果 originBehind == true，表示当前正在解析的 Block 落后于 L2ParentBlock.L1Origin
      1. l1Blocks 清空，理论上不可能出现。
3. BatchQueue.NextBatch
   1. AddBatch 里面最重要的是 CheckBatch（其中的 checkSpanBatch），保证了 batch 的数据一致性。
   2. 确定 l1Blocks[0] 或者 l1Blocks[1] 是 SpanBatch 起始的 L1Origin。
   3. SpanBatch 的第一个 Block 时间戳和最后一个 Block 的时间戳，需要包含 L2ParentBlock.Time + cfg.BlockTime。
   4. 调整 L2ParentBlock，通过 SpanBatch 的第一个 Block 时间戳 小于 L2ParentBlock 多少秒，L2ParentBlock 就向前多少。
      1. **处理冗余的情况，SpanBatch 内部前面部分的 Block 有冗余。一些 Block 被重复提交。**
   5. SpanBatch 开始的 L1Block 距离当前正在解析的 L1Block，差距不能超过 SeqWindowSize（14400-12小时）。
   6. SpanBatch 结尾的 Block 必须在 l1Blocks 并且 Hash 匹配。
   7. tx check(L1Origin 与 L2Blcok 时间戳差距 1800，应该没有tx 等等)
4. 如果 originBehind == true，表示当前正在解析的 Block 落后于 L2ParentBlock.L1Origin
   1. 此时 AddBatch 完成表示有新的数据进来了，此时可以直接返回 err（NotEnoughData/EOF）。
   2. 理论上不太会出现这种情况，因为 L2ParentBlock 来自全局的 L2SafeBlock 他都被派生过了，它的 L1Origin 还没有被解析，应该是内部一些异常引起的。
5. deriveNextBatch
   1. 遍历 batches 走上述 checkSpanBatch 逻辑，删除无效的 checkSpanBatch。
   2. 用第一个 checkSpanBatch 去生成 SingularBatch 用于返回。
      1. 其中 当前解析的 L1Block - SeqWindowSize（14400） 大于 l1Blocks[0].Number，返回 empty batch。

总结：
1. 这里面的校验逻辑较为细碎，但是他处理了 L2Block 冗余的场景。
2. L2Block 不会出现 gap，那样事无法派生的，是严重错误，op-batcher 可以提交重复但是不会遗漏，这是系统底线。
3. L2Block 可能提交分叉的数据，这部分在[op node reset](./7_op_node_reset.md) 部分。
4. 最重要的是，BatchQueue 返回了一个紧随在 L2ParentBlock 后面的 L2Block。
5. **L1Block 跨度（正在解析的 L2Block.L1Origin 与 正在读数据的 L1Block之间）超过 SequencerWindowSize（14400-12小时），则直接丢弃不再派生。**

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

AttributesQueue 会基于 pendingSafeHead 从 L1 中读出下一个 batch 然后组装成 AttributesWithParent。

### NextAttributes

1. BatchQueue.NextBatch，如果是管道的最后一个 SpanBatch，那么会设置 isLastInSpan，当派生时遇到这个变量 pendingSafeHead 会用于更新 safeHead。
2. SpanBatch 与 pendingSafeHead 校验 Number 和 Hash。
3. 调用 [FetchingAttributesBuilder.PreparePayloadAttributes](#preparepayloadattributes)
4. 修改 attrs.NoTxPool = true，对于派生时不会打包交易的所以需要设置一下，对于 Sequencer 就不需要设置了。