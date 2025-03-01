# op node reset

## Overview

op-node 在遇到 ResetErr 的时候会进行 Rest，让整个 op-node 回到一个可靠的位置，这个可靠是对于 L1 和 L2 都可考的位置。
自重主要的逻辑在 [EngineQueue.Reset](#enginequeuereset)，其余的部分都是根据这部分的结果（L1BaseBlock）进行 Reset。

## FindL2Heads

FindL2Heads 主要是推断出 L2FinalizedBlock，L2SafeBlock，L2UnSafeBlock，在正常情况下这三个变量都是由 op-node 
派发给 op-geth，在异常情况下主循环收到 ResetErr 的时候需要是的回归到一个可靠点继续派生或者出块，这种异常情况这下发的
结果可能不正确，所以需要重新根据当前 L1Chain 和 L2Chain 的情况进行推断。

推断的理论基础：
1. L2FinalizedBlock 是可靠的回退点。
2. 如果 L2FinalizedBlock 距离 L2SafeBlock 比较远，那么在 [L2FinalizedBlock, L2SafeBlock] 之前其 L1Origin + SequencerWindowSize(14400-12小时) < L2UnSafeBlock（其 L1OriginHash 是正确的） 并且是 Epoch 的第一个 L2Block。
3. 基于以上两点，验证 L2UnSafeBlock -> L2FinalizedBlock 的正确性。
4. 验证期间有任何异常，L2UnSafeBlock--，L2SafeBlock 在结束时回退到 L2FinalizedBlock。 
5. L2FinalizedBlock 太远或者不存在第二点作为保障。

推断过程：
1. 验证 L1Block：GetL1OriginBlock，与当前遍历的 L2Block 记录的 Hash 进行比对
   1. 比对失败，说明 L1 由 reorg
   2. 此时 L2UnSafeBlock 回退到 L2SafeBlock
   3. GetL1OriginBlock 的方式在第一次或者比对使用用的是 L1BlockRefByNumber，低于时刻就是 L1BlockRefByHash
   4. 理论上如果在主链上除了第一次之外都在 L1BlockRefByHash，否则就是 reorg
2. 验证 L2Block：L2BlockRefByHash 获取当前遍历的 L2ParentBlock
   1. 分是否跨 epoch，比对 hash，number，SequenceNumber
   2. 此处没有比时间戳，因为是获取的 parent，间隔 1s 是有保障的
   3. 验证成功之后 result.Safe 被设置为当前验证 L2Block，即 result.Safe 作为游标进行回退
3. **如果设置了 SkipSyncStartCheck，会跳过 L2UnSafeBlock -> L2SafeBlock 的验证，直接验证 L2SafeBlock -> L2FinalizedBlock 段**
   1. 在 L2UnSafeBlock -> L2SafeBlock 的 reorg 就会被默认是没有的
   2. 这个配置对于实战是比较有用的，因为实际情况 L2Chain 没有 reorg，并且多数情况下 L2SafeBlock == L2FinalizedBlock（原因见 [Finalizer](./2_op_node_components.md#finalizer)），所以效率很好
   3. **SkipSyncStartCheck 适合在同步数据期间重启，避免了大量的递归校验，追到最新块建议重启去掉，因为出块期间理论上 L2UnSafeBlock -> L2SafeBlock 这段存在 reorg**

### L1 Reorg

验证 L1Block 期间，如果 L1BlockRefByNumber 返回的结果与 L2Block.L1Origin 记录的不一致，这个时候就是 L1 发生了 reorg，L2UnSafeBlock-- 进行回溯。

### L2 Reorg

验证 L2Block 阶段保证 L2 是连续的，且回溯到 L2FinalizedBlock，说明 L2UnSafeBlock -> L2FinalizedBlock 在一条链上。
op-node 在 Reset 阶段是无法处理 L2 Reorg 的，因为他只能回溯保证 L2FinalizedBlock，L2SafeBlock，L2UnSafeBlock 在一条链上。

如果不在，这个时候会返回 TemporaryError，让主循环等一段时间，让 op-geth 去处理它的 fork。

## EngineQueue.Reset

EngineQueue 是整个 Reset 的核心部分，会根据 FindL2Heads 返回结果设置 [EngineController.MetaData](2_op_node_components.md#metadata)。
然后根据 L2SafeBlock 获取其对应的 L1Origin，以此作为 base reset 其余的 [derivation data](./4_op_node_derivation_data.md) 数据模块。

## ResetErr

前面介绍了 Reset 的主要思路和做法，接下来我们看一下什么情况线会返回给主循环 ResetErr。

1. EngineController 
   1. ConfirmPayload 中返回的结果无法组装成 L2Block（Schema 错误）。
   2. ForkchoiceUpdate 更新的 MetaBlock 在 op-geth 中没有，导致设置失败。
2. AttributesHandler
   1. Proceed，EngineQueue 处理的 attributes 与 PendingSafeL2Head 不匹配，此时 SafeL2Head 这趟线不连续了。
   2. consolidateNextSafeAttributes，PendingSafeL2Head + 1 NotFound 或者返回的数据无法解析成 L2Block。
      1. 这个函数要就是处理分叉，以传进来的 attributes 为准，但是需要从 L2 获取对应的 Block 进行比对。
      2. 这个函数就是在 PendingSafeL2Head 与 UnafeL2Head 差距较大的时候调用，处理分叉的情况，所以 PendingSafeL2Head + 1 必须有。
   3. forceNextSafeAttributes，派生过程中 op-geth 返回 BlockInsertPrestateErr，标识 parent state 没有。
3. AttributesQueue
   1. NextAttributes.createNextAttributes 等待解析的 batch 的 Hash 与时间戳需要与 EngineQueue 给的 L2ParentBlock 进行校验。
   2. BatchQueue.NextBatch.deriveNextBatch EngineQueue 给的 L2ParentBlock 与本次缓存的 l1Blocks[0] 进行校验，表示派生数据的延续性。
4. EngineQueue
   1. verifyNewL1Origin，更新正在解析的 L1Origin 到本地 EngineQueue.Origin 进行校验失败
      1. newOrigin 与 unsafeOrigin.Number 或 unsafeOrigin.Number + 1 相等，就比较他们的 Hash，大多数情况他们应该是相邻的。
      2. 否则，就比对 L1Chain 上 unsafeOrigin 与本地的情况，防止 L1 分叉。
   2. Reset 期间的异常。
5. Finalizer
   1. tryFinalize 确认 SafeBlock 要确认 finalizedL1（最近的） 和 finalizedDerivedFrom（派生L1Block）是否都是在在上。
6. FetchingAttributesBuilder
   1. PreparePayloadAttributes，l2Parent.L1Origin 与 L1Epoch 进行校验。
   2. PreparePayloadAttributes，nextL2Time 必须大于等一 L1EpochTime。
7. Sequencer
   1. StartBuildingBlock，开始出新块，[L1OriginSelector.FindL1Origin](./1_op_node_clients.md#findl1origin) 后去的 L1Epoch 与 UnsafeL2Head.L1Origin 进行校验。
8. L1Traversal
   1. AdvanceL1Block 获取下一个 L1Origin 作为基准派生，但是会和前一个做 hash 校验。

## OtherErr

了解了 ResetErr 之后，我们也看一下主循环还会接受到什么其他类型的 Err，看一下他们在什么情况下会被返回，以及怎么处理。

### ErrCritical

直接返回。

主要是数据格式错误，TX 解析错误，EngineAPI 返回不识别的错误类型，升级 TX 的错误等会返回。

### EngineELSyncing

没有操作。

在派生开始之前判断是否是在 ELSync，是的话就返回这个错误只有一种情况。

### ErrTemporary

等待本次操作剩余时间。

主要是一些因为网络请求失败，可能等一下就会正常的情况会返回这个错误。

### NotEnoughData

等待本次操作剩余时间。

[derivation data](./4_op_node_derivation_data.md) 部分需要的数据还没到或者目前没有数据提供，等一下就一般就可以了。
