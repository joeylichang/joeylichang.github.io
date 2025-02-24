# op-node components

## EngineController

### Overview

EngineController 是 op-node 通过 EngineAPI 控制 op-geth 的模块，是 op-node 实现上的基石模块，上层的逻辑模块通过它完成对 op-geth 的控制。
EngineController 主要分为三组 API。
1. `SetUnsafeHead`，`SetBackupUnsafeL2Head`，`SetSafeHead`，`SetPendingSafeL2Head`，`SetFinalizedHead`
    1. 上层逻辑控制设置 op-geth 的一些元数据，这些元数据仅仅存储在 EngineController 内，等待其他流程触发的时候下发给 op-geth。
2. `StartPayload`，`ConfirmPayload`，`CancelPayload`，`TryUpdateEngine`
    1. 这些接口内部都调用了 EngineAPI，当然有 EngineController 自己的一些逻辑，这些接口也主要是受上层模块的控制。
    2. 其中前三个个接口理论上只有在 `CL` 模式下使用，或者说 `EL` 同步完毕，但是只有 `StartPayload` 做了强校验，另外两个要么靠上层逻辑，要么靠 op-geth 来考虑这种情况。
3. `InsertUnsafePayload`，`TryBackupUnsafeReorg`
    1. 这俩个接口相对复杂一些，前者要考虑 `EL` 和 `CL` 两种模式，看着代码像是在原有逻辑上后区分两种模式，所以比较难懂一些。
    2. 后者与 reorg 有关，需要结合上层的逻辑继续理解。

要想更好的理解 EngineController，除此之外还需要理解 [EngineController MetaData](#metadata) 和 [ELSyncMode and CLSyncMode](#elsyncmode-and-clsyncmode)。

### StartPayload

1. not `IsEngineSyncing`.
2. if `buildingInfo` is mot empty, will overhead the prev building payload.
3. call `startPayload` with `Attributes` and newest `ForkchoiceState`.
4. update `buildingInfo` info:
    1. `buildingInfo = PayloadInfo`
    2. `buildingSafe = updateSafe`, 来自参数，上层的控制逻辑.
    3. `buildingOnto = parent`, 当前正在出块的前一个块.
    4. `if updateSafe, e.safeAttrs = attrs`

**注意**
* 这里需要理解一下 `updateSafe` 参数的含义，仅仅在派生阶段会被设置为 true，因为派生的数据来自 L1，来自 L1 的数据有一个安全距离（目前配置的是 15），确保 L1 的数据来自主链。
但是这职能说明 L1 的数据不变了，L2 还存在回滚的可能，因为 L2 可能将 fork 的数据也提交到了 L1， 上层的 [AttributesHandler](#attributeshandler) 会处理这部分的逻辑。
* buildingXXX 表示正在执行的信息，并且只有在 StartPayload 内进行设置，所以无论派生还是出新块的逻辑都会先调用看他们的信息，决定上一次的任务是否完成等等才能新的任务。

### ConfirmPayload

1. if e.combinedAPI && !e.buildingSafe，前者需要配置 `--sequencer.combined-engine`。
   1. 不能是来自派生，这个特性就是给 sequencer 用的。
   2. confirmPayloadCombined -> [SealPayload](./3_op_node_engine_api.md#sealpayload)。
2. 没有配置或者是派生。
   1. 数据优先从 agossip 获取，然后从参数获取。
   2. 做一些检查，没问题通过 agossip 广播出去。
   3. [NewPayload](./3_op_node_engine_api.md#newpayload)，下发给 op-geth 并插入到 chain 中。
   4. [ForkchoiceUpdate](./3_op_node_engine_api.md#forkchoiceupdate)，根据上面返回的结果设置元数据，并在此下发给 op-geth。
3. 更新元数据
   1. if unsafeHead.Number >= ref.Number，e.SetBackupUnsafeL2Head(e.unsafeHead, false)
      1. 标识新的 block 比 unsafe 小，其实是一次 reorg，需要记录 fork point，便于后续需要的回滚，
   2. unsafeHead = ref，更新 unsafeHead
   3. if buildingSafe，pendingSafeHead = ref ，来自派生需要移动 pendingSafeHead。
   4. if buildingSafe && e.safeAttrs != nil && e.safeAttrs.IsLastInSpan，主要是 IsLastInSpan ，
      1. safe = ref，更新 safe
      2. SetBackupUnsafeL2Head(eth.L2BlockRef{}, false)，擦出之前的 fork point。
4. resetBuildingState

### CancelPayload

只是简单的 GetPayload 查询一下，然后 reset buinding 信息。这里为什么查询一下就可以呢？
因为 op-geth 即使有这个错误数据也没关系，也可以从从前的 block 开始新的任务生成 block。 
op-geth 的元数据（unsafe/safe/finalized）都是来自 op-node，op-geth 即使存储了多余
的垃圾数据也无妨，只要元数据控制的正确就不会访问到。

### TryUpdateEngine

根据目前的元数据，向 op-geth 推送。即使是 ELSyncMode 也会推送，因为 op-geth 会根据这些元数据控制 P2P 的同步。

### InsertUnsafePayload

InsertUnsafePayload 处理来自 P2P 的数据，并在在进入这个之前是做了校验的了，一定是在 unsafe 之后，即表示这个数据应该是在当前分支之后的。

#### CLSync

1. status, err := NewPayload
   1. err == "forced head needed for startup" , 忽略这个错误，后面会处理
      1. 这个错误是 op-geth 返回的，出现的原因只有一个就是 ParentBlock 找不到，这个时候需要重新 reset，后面会处理
      2. 这个错误可能会引起 [ELSync](#elsync)
   2. 其他错误会直接返回 TemporaryError
2. status == eth.ExecutionInconsistent
   1. 这个个错误是 op-geth 返回的，出现的原因只有一个就是 pathdb 模式下，没有找到 ParentState
   2. trySyncingWithEngine 尝试 5 次 ForkchoiceUpdate
      1. 本地缓存的 unsafe，safe 和 finalized
      2. 其实就开始了 [ELSync](#elsync)，但是 op-node 没有状态的转换。
3. ForkchoiceUpdate
   1. 参数传进来的 unsafe
   2. 本地缓存的 safe 和 finalized

#### ELSync

1. 如果 syncStatus == syncStatusWillStartEL
   1. 表示初始状态，此时需要判断是否开始
      1. 注意 syncStatusWillStartEL 状态只有在启动的时候设置，一旦 ELSync 结束之后是不会在又这个状态了
   2. 获取 L2FinalizeBlock 和 L2UnSafeBlock
   3. 如果 参数 ref 与 L2UnSafeBlock 差距大于 86400（24小时） || L2FinalizeBlock 就是创世纪块 || 获取 L2FinalizeBlock 返回 error
      1. 此时应该开启新的一轮 ELSync，syncStatus = syncStatusStartedEL
      2. 否则，如果获取 L2FinalizeBlock 返回 error == nil，syncStatus = syncStatusFinishedEL 关闭 ELSync
      3. 否则，返回 TemporaryError
2. status, err := NewPayload
   1. err == "forced head needed for startup" , 忽略这个错误，后面会处理 ==> 同 CLSync
   2. status == eth.ExecutionInconsistent || checkELSyncTriggered(status.Status, err)
      1. 如果满足这个条件无论是 CLSync 还是 ELSync 都会尝试 ELSync
      2. trySyncingWithEngine 尝试 5 次 ForkchoiceUpdate
         1. 本地缓存的 unsafe，safe 和 finalized
      3. ForkchoiceUpdate 通知 op-geth 元数据之后，本次存储数据的 gap 会通过 op-geth 的 P2P 自行同步。
      4. checkELSyncTriggered 出发条件必须同时满足如下条件：
         1. err == "forced head needed for startup"
            1. ParentBlock 找不到
         2. status == eth.ExecutionSyncing
            1. 在 NewPayload 中 "forced head needed for startup" 与 ExecutionSyncing 会同时被返回。
         3. e.syncMode != sync.ELSync
3. checkNewPayloadStatus
   1. status == eth.ExecutionValid && e.syncStatus == syncStatusStartedEL
   2. syncStatus = syncStatusFinishedELButNotFinalized
4. syncStatus == syncStatusFinishedELButNotFinalized
   1. 更新本地 SafeHead 和 FinalizedHead
5. ForkchoiceUpdate
   1. 参数传进来的 unsafe
   2. 本地缓存的 safe 和 finalized
6. checkForkchoiceUpdatedStatus
   1. status == eth.ExecutionValid && e.syncStatus == syncStatusStartedEL
   2. e.syncStatus = syncStatusFinishedELButNotFinalized
7. checkUpdateUnsafeHead
   1. 更新本地 UnSafeHead
8. syncStatus == syncStatusFinishedELButNotFinalized
   1. syncStatus = syncStatusFinishedEL，结束了 ELSync

#### 总结

* CLSync
  1. InsertUnsafePayload 就是在末尾插入一个 Block，忽律 block 和 state 不存在的错误，用最新的元数据去更新 op-geth。
  2. 至于元数据更新之后和 op-geth 时间存储的数据（block 和 state）之间的 gap，那就要靠 op-geth 的 P2P 去同步，不再是 op-node 关心的。

* ELSync
  1. 启动方式有两种，一种是启动的时候指定了 ELSync 模式但是他只能在运行期间执行一次 ELSync（也是需要 Gap 超过 86400 个 Block，即24小时）。
  2. 另一种是 NewPayload 返回的状态和错误明确了，插入的 Block 数据（block/state 数据不存在 或者 downloader 层面返回需要重新 update 元数据），这个时候会执行新的元数据，op-geth 开始通过 P2P 同步大量的数据。
  3. NewPayload 或者 ForkchoiceUpdate 成功，则会更新为 syncStatusFinishedELButNotFinalized。
  4. 在函数的最后会变为 syncStatusFinishedEL。

### TryBackupUnsafeReorg

TryBackupUnsafeReorg 主要作用就是根据记录的 backupUnsafeHead ，向 op-geth 发送 ForkchoiceUpdate 强制设置其元数据进行回滚。
backupUnsafeHead 记录了上次 fork block，在 AttributesHandler 派生的过程中发生了回滚就会设置标志为，在每一轮派生和 sequencer 开始之前都会先执行它来进行回滚。

逻辑上没有什么特别的，只是需要注意一点，在 ForkchoiceUpdate 成功之后会 SetUnsafeHead(BackupUnsafeL2Head())，即**重启调整 unsafeHead，但是 safe 和 finalize 不会，因为他们俩一旦确认就不会改变了**。

### MetaData

EngineController 中维护着一下重要的元数据，这些元数据决定了最终 L2 Unsafe/Safe/Finalize Block，他们配合着上层的逻辑完成最终的确定，所以需要对他们展开的描述一下。

* safeHead

理论上只有在一种情况下，L2Block 会被设置成 Safe，即在 ConfirmPayload 完成之后且满足：
`e.buildingSafe && e.safeAttrs != nil && e.safeAttrs.IsLastInSpan`
其中，`e.buildingSafe` 仅仅在派生的流程中会被确认是 Safe（StartPayload 接口传进来）时而设置，在 Sequencer 的流程中不会被确认（[op_node](./op_node.md) 有介绍）。即，派生阶段的 StartPayload 都是 safe 的，Sequencer 都不是 Safe 的。
`e.safeAttrs` 也是 StartPayload 接口传进来的，`IsLastInSpan` 可以在 [op_node_derivation_data](./4_op_node_derivation_data.md) 中理解。

**注意: 仅仅在派生阶段 Safe 才会被设置，其他时候都不会。**

其他情况下设置 safeHead 都是规整的需要：
1. derivation 被 reset 的时候会设置 Safe。
2. InsertUnsafePayload 阶段，发现需要出发 EL Sync，并且 L2 返回的 Meta 有异常需要更新 Safe 或者 Finalized
    1. 异常：L2Safe.Number > L2Unsafe.Number，需要更新 Safe，L2Finalized.Number > L2Unsafe.Number，需要更新 Finalized。
        1. 更新之后 op-geth 会通过 P2P 进行同步（EL Sync 启动）。
    2. InsertUnsafePayload 节点，EL sync完成但是没有 Finalized（`syncStatusFinishedELButNotFinalized`），需要设置 Safe 和 Finalized 为 `ForkchoiceState` 做准备。
3. AttributesHandler 处理 reorg 的时候会回溯正确的 block 并设置 pendingSafeHead，如果 IsLastInSpan，会设置 safeHead。


* pendingSafeHead

pendingSafeHead 理论上应该是等于 unsafeHead 他作为游标在一定条件下更新 safeHead，这个条件就是 `safeAttrs.IsLastInSpan`。
只有在派生阶段才会更新 pendingSafeHead，并同时满足 IsLastInSpan 才会更新 safeHead。

**注意：Span 像是被提交到 L1 的一批 Block 的单位，只有是最后一个能确认才说明没有遗漏提交或者说是一种抗审查。**

其他情况下设置 pendingSafeHead 都是异常情况修复的需要：
1. derivation 被 reset 的时候会设置为 Safe。
2. AttributesHandler 处理 reorg 的时候会回溯正确的 block 并设置 pendingSafeHead，如果 IsLastInSpan，会设置 safeHead。
3. AttributesHandler 派生的过程中（StartPayload + ConfirmPayload）返回 `BlockInsertPayloadErr` 类型错误时候，回滚到 Safe，即设置 pendingSafeHead 为 pendingSafeHead。

**注意：`BlockInsertPayloadErr` 这个错误值得讨论一下，他表示插入的 Block 有问题，比如 StateRoot 校验或者执行失败等，这个时候是需要回滚的。会通过设置 backupUnsafeHead 并设置标志为强制回滚。**

* backupUnsafeHead

`BlockInsertPayloadErr` 这个错误值得讨论一下，他表示插入的 Block 有问题，比如 StateRoot 校验或者执行失败等，这个时候是需要回滚的。
这个时候处理会设置 pendingSafeHead 到 safeHead，还会通过设置标志为强制回滚到 backupUnsafeHead（这个标识设置之后，在下一轮派生或者定序之前会进行回滚）。

那么这个标识职位什么时候会被设置呢？这个标识位在成功派生（ConfirmPayload）之后，并且派生的 BlockNumber 大于当前记录的 unsafeHead.Number ，说明有回滚了，backupUnsafeHead 被设置成 unsafeHead（然后更新 unsafeHead 为新的 Block），用于记录前一个 fork，便于回滚。
其他的时候对 backupUnsafeHead 的设置都是为了擦出，即设置为 empty block。

所以 backupUnsafeHead 的作用是记录，记录前一个的 fork block，在 TryBackupUnsafeReorg 的时候会用它来 ForkchoiceState。

* unsafeHead

unsafeHead 就是比较好理解了，正常情况下控制完（ConfirmPayload） op-geth 出块就会更新。多数异常情况下也会更新，总结起来意义也不大，故不赘述。
但是有一点可以说一下，就是在 AttributesHandler 控制派生的过程中如果 pendingSafeHead 大于了 unsafeHead，这是一种异常情况，会回滚推进 unsafeHead 到 pendingSafeHead。

1. derivation 被 reset 的时候会被重置(来自 op-geth，后面详细介绍)。

* finalizedHead

[Finalizer](#finalizer) 里面会确认当前正在派生的 L1BLock 以及过去派生过的 L1Block 与 L2Block 的对应关系表，来确认 L1 的确定度，从而确定 L2 的 finalizedHead。
其他场景都是规整需要，不赘述。

#### 总结：
1. unsafeHead：标识最新派生的 L2Block。
2. backupUnsafeHead：标识最近一个 fork ，用于派生发生错误的时候进行回滚。
3. pendingSafeHead：只有在派生流程中会更新，在 sequencer 阶段不会，派生成功之后就会更新。
4. safeHead：在 lastSpan 的时候（详情见：[derivation data](./4_op_node_derivation_data.md)），pendingSafeHead 会用来更新 safeHead，pendingSafeHead 像是 safe 的游标。
5. finalizedHead：通过记录的历史 L1Block 和 L2Block 对照关系，确定 L1Block 稳定之后，更新对应的 L2 的 finalizedHead。

**注意：unsafeHead 用于 Sequencer 推进进度，pendingSafeHead 用于派生推进进度。**

### ELSyncMode and CLSyncMode

简单说 CLSyncMode 就是处理来自 L1 的数据通过 [DerivationPipeline](./5_op_node_derivation_pipeline.md) 进行的调用。
ELSyncMode 是让 op-geth 通过其自身的 P2P 进行同步数据，这个时候 op-node.EngineController 就是一个完全的控制器不负责数据流，
它只会告诉 op-geth UnSafe/Safe/Finalized 这些元数据，相当于同步的目标，至于过程中的数据流是不经过 op-node 的。ElSync 结束之后
后面会转入 CLSync，如果追数据的过程中发生了一些异常也还会在切换回 ELSync，但是这个切换是强制了无法配置，也没有状态转换，就是单纯的
通知 op-geth 元数据然后让其去运行。

这里面有两个问题需要搞清楚：
1. ELSyncMode 告诉了 op-geth 同步的目标，但是 op-geth 同步的结果并没有通知 op-node，那么 ELSync 怎么知道 op-geth 的进展，并停止 ELSync 转入 CLSync？
2. ClSyncMode 与 ELSyncMode 的关系是什么样的？有明确的界限吗？界限是什么？

首先，op-node 内存保存着 op-geth 的元数据，即目标位置。其次，ELSyncMode 状态转换必须经过 InsertUnsafePayload 接口，这个接口目前只有来自 P2P 的数据
（注意：这个 P2P 是 op-node 的 P2P），所以 ELSync 的开始和结束都是从 op-node P2P 开始的，在开始的时候通过 P2P 来自的数据，处理了 ELSync 状态机的转动，
P2P 网络一直运行，当发现了当前元数据的下一个 Block 数据的时候（op-node 主循环的逻辑）会在此调用 InsertUnsafePayload，如果前面的 ELSync 已经完成，这里就会
更新状态表示结束，后面切换为 CLSync，如果没有完成可能是将进度像后推进了（有了新的 Block）。

ELSyncMode 可以说就是通过 op-geth 的 P2P 网络追数据，这块服用了 geth 的逻辑，即 P2P 网络时刻都在同步数据，所以说即使是 CLSync，op-geth 自身也在同步，那么为什么
op-geth 与 op-node 之间的元数据没有差异呢？ 主要是 engine api 的兼容。如上所述，CLSyncMode 也会触发 op-geth 的 P2P 的 range sync。
所以，如果 ELSyncMode 表述的是 op-gteh 的 P2P 网络数据流，其实两者没有界限，但是在 op-node.EngineController 中的 ELSyncMode 只有一次，且完事之后转换为 CLSync，不在会切换回来。

## CLSync

CLSync 内部维护了一个队列（根据 L2BlockNumber 排序），存储 Payload，只有三个接口 `AddUnsafePayload`, `LowestQueuedUnsafeBlock`, `Proceed`，语义比较明确，重点是第三个。
`Proceed` 从队列中取出 L2BlockNumber 最小的 Payload 调用 `EngineController.InsertUnsafePayload`，其逻辑在 `CL` 模式下仅仅是调用 EngineAPI 的 `NewPayload` 和 `ForkchoiceUpdate` without attr.
所以，上层事先收到了 L2 的信息调用了 `AddUnsafePayload`（这部分派生部分会介绍）。

## AttributesHandler

AttributesHandler 是派生过程的核心逻辑，这里面根据数据源 AttributesQueue 来的 Attributes 数据来控制 EngineController 完成 L2 的派生。
最主要的是这里完成了 L2 reorg 的处理。Attributes 数据在每一轮派生的结尾会通过 SetAttributes 设置给 AttributesHandler，在下一轮派生的时候调用 Process 完成派生，包括 reorg 的处理。

**注意：这里处理的 reorg 并不是来自 L1 的 L2 数据带有 forks，这部分在 AttributesQueue 内部解决，这块解决的 reorg 是当前 op-geth 执行的过程中收到的区块和派生之间的 reorg。**

### Proceed

1. 基础判断
    1. `ec.PendingSafeL2Head() != attributes.Parent`，要处理的数据不是前一个处理过的尾部，里面会判断 attributes 月 pendingSafe 是不是同一个数据，不是就返回 error。
2. 到此，说明即将处理的数据是前面 PendingSafeL2Head 的下一个 block，所以下面需要判断之前的 PendingSafeL2Head 是否是最后一个 block，即 UnsafeL2Head：
    1. `PendingSafeL2Head().Number < ec.UnsafeL2Head().Number`，派生的 block 不是最新的 block，接下来要在前一个派生的结果上继续派生，这就需要解决 fork 问题。
        1. `consolidateNextSafeAttributes`
            1. 这个时候 UnsafeL2Head 大于 PendingSafeL2Head 说明，派生的不是最长链
            2. 这样我们获取搭配 PendingSafe + 1 的 Block，进行验证，如果验证通过，仅仅说明派生的慢了，但是数据没有问题，更新 PendingSafeL2Head （如果是 IsLastInSpan 还需要更新 SafeHead）即可。
            3. 如果验证不通过，那就就那当前的 attributes 数据强制派生 `forceNextSafeAttributes`。
            4. 注意：**这就说明数据源给的数据是一定正确的，处理了 L2 的 fork 数据**
    2. `PendingSafeL2Head().Number == ec.UnsafeL2Head().Number`，这是符合预期的，只需要正常派生就可以了
        1. `forceNextSafeAttributes`
            1. ec.StartPayload + ec.ConfirmPayload
            2. 如果是 BlockInsertPayloadErr 错误，那就是明确的派生的 block 有问题，比如 StateRoot 验证或者交易执行失败等。
                1. `ec.SetPendingSafeL2Head(ec.SafeL2Head())`，回滚 PendingSafe 到 Safe。
                2. `ec.SetBackupUnsafeL2Head(ec.BackupUnsafeL2Head(), true)`，设置回滚标志位，会在下一轮派生的时候 TryBackupUnsafeReorg。
    3. `PendingSafeL2Head().Number > ec.UnsafeL2Head().Number`，这一种非预期的异常情况，这轮派生仅仅会重置元数据，但不会继续了
        1. `ec.SetUnsafeHead(ec.PendingSafeL2Head())`

## Finalizer

Finalizer 的作用就是确定 L2FinalizedBlock，它的确认方法是先确定 L1DerivationBlock，然后保存 L1DerivationBlock 与 L2SafeBlock 的对应关系（finalityData），保存历史 129个 版本。
然后确认当前和 L1FinalizedBlock 距离最近的 L1DerivationBlock，然后在历史中找到 L1DerivationBlock 对应的 L2SafeBlock 作为 L2FinalizedBlock。

这里需要解释一下 L1DerivationBlock，它描述的是当前正在派生的 Block，并不是 L1Origin。
* L1Origin：表示 L2 依赖其配置和 Deposit Txns 进行派生的那个 block。
* L1DerivationBlock：标识当前正在解析的 block，当前的 L2 数据来自这个 block，即 op-batcher 提交的 Block。

finalityData 表示在当前 L1DerivationBlock 派生了 L2PendingSafeBlock 的时刻的 L2SafeBlock（注意并不是当前派生的最新 L2PendingSafeBlock 或者 L2UnSafeBlock）与 L1FinalizedBlock 的映射关系的 129 个版本，
即在当前派生block 的情况下经过处理之后的 L2SafeBlock，选择一个最接近 L1FinalizedBlock 的 L2SafeBlock 作为 L2FinalizedBlock。

**注意**：假设每个派生的 L2Block 都是 IsLastSpan，那么每次派生之后 L2PendingSafeBlock = L2SafeBlock，finalityData 总能记录到最新的 L2SafeBlock 与 L1DerivationBlock 的映射关系，
如果 L1FinalizedBlock 更新及时那么 L1FinalizedBlock 永远保持与最新的 L1DerivationBlock 相等，那么 L2SafeBlock 总是等于 L2FinalizedBlock。
在或者 L2Block 不都是 IsLastSpan，只是 Span 不大，一样可以保证 L1DerivationBlock 对应到最新的 L2SafeBlock ，此时 L2SafeBlock 与 L2UnSafeBlock 的差距完全取决于 op-bathcer 提交的时间将额。
这正是目前线上看到的情况。

### PostProcessSafeL2

输入 L1DerivationBlock 和 L2SafeBlock，存储在 finalityData 内部，这里有两点注意：
1. finalityData 只保存 129 个版本，多余的从前面益处。
2. L1DerivationBlock 永远大于等于 finalityData 记录的 L1 Block，因为这个 L1Block 代表的是 Finalized，目前是距离最新的 15个一定是递增的。
3. L1DerivationBlock 相等的话，只需要更新 L2SafeBlock 的信息就可以，我们只需要保留有效值。

### Finalize

收到来自 L1 的 L1FinalizedBlock 更新本地信息，然后尝试 tryFinalize。

### OnDerivationL1End

派生完成之后判断是否需要 tryFinalize，标准呢是将额 15 个 L1Block。

`tryFinalize`
1. 在 finalityData 中找到 L2SafeBlock 大于前一个 L2FinalizedBlock 并且记录的 L1DerivationBlock 小于等于当前 L1FinalizedBlock。
2. 在 L1 上查找相应的 L1DerivationBlock 和当前的 L1FinalizedBlock 进行校验 hash ，标识两个都是有效的，此时证明 L1 的这条了解没问题，那么对应的 L2SafeBlock 就没问题，可以设置为 L2FinalizedBlock。

**注意：Finalizer 里面获取 L1 信息全部使用的 [confDepth](./1_op_node_clients.md#confdepth) 距离最新块有一个安全距离。**