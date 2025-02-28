# op-node sequencer

## Overview

Sequencer 是 op-node 的定序模块，用于控制 op-geth 出块，也是通过控制 [EngineController](./2_op_node_components.md#enginecontroller) 来完成的，
所以 EngineController 接受来自 Sequencer 和 DerivationPipeline 两个重要逻辑的指令，协调完成对 op-geth 的控制。

Sequencer 内部有两个重要的成员变量 [FetchingAttributesBuilder](./4_op_node_derivation_data.md#fetchingattributesbuilder) 和 [L1OriginSelector](./1_op_node_clients.md#l1originselector)。
FetchingAttributesBuilder 用于从 L1 获取数据生成 PayloadAttributes 模板供 op-geth 往里面打包交易，模板中主要包括一些 L1 的基础数据，以及三类交易（系统配置的 deposit 类型交易，用户的 deposit 交易，升级交易）。
L1OriginSelector 需要根据目前的 L2Block 信息，获取下一个 L2Block 对应的 L1Origin（保证 L1 停止了 30min 以上 L2 也会停止），所以 Sequencer 的 L1 客户端不是普通的客户端，在出块阶段会使用 [FindL1Origin](./1_op_node_clients.md#findl1origin) 方法来获取 L1Origin。

## StartBuildingBlock

1. l1OriginSelector.FindL1Origin(l2Head) => l1Origin
2. l2Head.L1Origin.Hash == l1Origin.ParentHash || l2Head.L1Origin.Hash == l1Origin.Hash
   1. 下一个 L2Block 可能没换 L1Origin 或者换为 L1Origin + 1
3. attrBuilder.PreparePayloadAttributes(l2Head, l1Origin.ID()) => attrs，生成 PayloadAttributes 模板
4. attrs.NoTxPool = uint64(attrs.Timestamp) > l1Origin.Time+d.spec.MaxSequencerDrift(l1Origin.Time)
   1. 如果 L1Origin 的时间超过 attrs 时间的 MaxSequencerDrift == 1800 = 30min，这个时候就不允许打包交易了，相当于必须是空块，加快进度追块 L1 的进度（可能 L1 停了，或者请求 L1 网络有问题）。
5. withParent := &derive.AttributesWithParent{Attributes: attrs, Parent: l2Head, IsLastInSpan: false} => StartPayload(ctx, l2Head, withParent, false)
   1. AttributesWithParent.IsLastInSpan 在这里被设置成 false ，其实在 StartPayload 中没有用到它，这里就是站位用，IsLastInSpan 只能是 op-bathcer 打包交易的时候使用，op-node 只有在解析的时候遇到，出块场景是不会用到的。
   2. StartPayload 的最后一个参数 `updateSafe` 只有在派生的时候才会为 true，对于 Sequencer 来说可能会回滚所以不会设置，设置了之后对 [EngineController.MetaData](./2_op_node_components.md#metadata) 有影响。

## PlanNextSequencerAction

PlanNextSequencerAction 用于计算 [RunNextSequencerAction](#runnextsequenceraction) 每次调用所需要的时间，即在调用 RunNextSequencerAction 之前，需要调用 PlanNextSequencerAction 来计算时间并设置定时器，到了之后可以下一次调用 RunNextSequencerAction。

1. buildingOnto, buildingID, safe := d.engine.BuildingPayload()，返回 [EngineController](./2_op_node_components.md#enginecontroller) 正在运行的数据。
   1. buildingOnto 表示正生成或者派生的 L2Block 的 L2ParentBlock。
   2. buildingID 表示当前正生成或者派生的 PayloadID。
   3. safe 表示是否是来自派生。
   4. 这三个变量只有在 [EngineController.StartPayload](./2_op_node_components.md#startpayload) 的时候被设置，在结束或者异常的时候被擦出。
2. if safe, return 1s
   1. 如果正在运行的派生来的数据，需要给充足的时间。
3. d.nextAction.Sub(now) > 0 && buildingOnto.Hash == engine.UnsafeL2Head().Hash
   1. 正在生成的 L2Block 是在 UnsafeL2Head 之后，表示正常出块中，且有剩余时间，就返回剩余时间。
4. buildingID != (eth.PayloadID{}) && buildingOnto.Hash == engine.UnsafeL2Head().Hash
   1. 表示正在构建，并且构建的信息是正确的，也就是说没有异常
   2. payloadTime.Sub(now) - sealingDuration(1ms) 或者 0，返回剩余的构建时间
5. 如果没有构建或者构建信息不对
   1. 应该立刻返回 0 ，立刻开始下一次的构建。
   2. **也有一种情况特殊，前一个 block 出块快了，现在剩余的时间大于 blocktime 1s，需要返回这个差值用于 sleep，这样 L1 和 L2 才能对齐**

nextAction 的作用是标记下一个动作的时间，是在 RunNextSequencerAction 里面更新，基本的原则是：
1. 返回的 ErrReset，nextAction = now + BlockTime（1s）
2. 返回去其他 Err，nextAction = now + 1s
3. 正确返回没有更新这个时间

PlanNextSequencerAction 总结：
1. 如果是派生（更新 safe head），返回 BlockTime（1s），给他足够的时间更新。
2. 如果正在构建的 block 是延续的（没有分叉），并且 nextAction 有剩余时间，直接返回这个剩余时间
3. nextAction 没有剩余时间，并且正在构建的 block 是延续的（没有分叉），返回 payloadTime - now -1ms（小于 0 则返回 0）
4. 否则，返回 payloadTime - now - BlockTime（1s）(小于 0 返回 0)
   1. 任务有异常，那么是给剩余的时间（此时也没有立刻结束，应该是给一些时间收尾）。
5. **注意：如果 payloadTime 剩余时间不足，此时返回的时间是0，也就是 L2 停止了一段时间之后，会连续出空块。**

## RunNextSequencerAction

RunNextSequencerAction 开始新的区块构建，或 seal 现有任务，最好先等待 PlanNextSequencerAction 返回的延迟后再执行。如果新区块成功密封，则将返回该区块以供发布；如果未密封，则返回 nil。
注意这里的错误处理哦！

1. onto, buildingID, safe := d.engine.BuildingPayload() 
2. buildingID != (eth.PayloadID{}) || agossip.Get() != nil
   1. 前者表示有数据正在处理，后者表示又来自 P2P 的数据可以处理不用自己生成。
   2. CompleteBuildingBlock => [EngineController.ConfirmPayload](./2_op_node_components.md#confirmpayload)。
   3. if safe
      1. d.nextAction + 1s
   4. err == nil
      1. attrBuilder.CachePayloadByHash
   5. err == ErrCritical
      1. return err
   6. err == ErrReset / other
      1. CancelBuildingBlock 
      2. d.nextAction + 1s
   7. err == ErrTemporary 
      1. d.nextAction + 1s
3. 如果没有数据待处理：
   1. [StartBuildingBlock](#startbuildingblock)
   2. err == nil, return
   3. err == ErrCritical
      1. return err
   4. err == derive.ErrReset / other /ErrTemporary
      1. d.nextAction + 1s

总结：
1. 继续前面没完成的任务，确认一下
2. 否则，开启新的构建任务