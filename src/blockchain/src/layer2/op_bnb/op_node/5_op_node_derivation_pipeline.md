# op-node derivation pipeline

## Overview

DerivationPipeline 是解析 L1 的数据派生 L2 的过程，也是 op-node 最核心的流程之一，如果是没有开启 sequencer 模式，derivation 就是唯一的核心逻辑了。
DerivationPipeline 内部分为两大部分：
1. AttributesQueue：可以理解为是数据部分，解析来自 L1 的数据，见 [derivation data parser](./3_op_node_derivation_data.md)。
2. EngineQueue：运行逻辑部分，内部通过 Finalizer 和 AttributesHandler 驱动 EngineController 完成对 op-geth 的控制。
   1. AttributesHandler：从 AttributesQueue 获取数据，然后进行检查，校准（解决 reorg 等）然后通过 EngineController 完成 `StartPayload`，`ConfirmPayload` 的派生流程。
   2. Finalizer：校准并确认 L2 的 Finalize。
   3. 对于 EngineQueue 来说 AttributesQueue 总是能够给出下一个 Attributes。

## Core

### EngineQueue.Step
1. [EngineController.TryBackupUnsafeReorg](./1_op_node_components.md#trybackupunsafereorg)
   1. 先处理 [AttributesHandler.Proceed](./1_op_node_components.md#proceed) 派生阶段发现的 reorg。
   2. 详情见链接，概括的说就是调用了 ForkchoiceUpdate 更新了 op-geth 的元数据。
2. [EngineController.TryUpdateEngine](./1_op_node_components.md#tryupdateengine)
   1. 强制规整一下状态（如果需要的话，里面有判断是否需要更新）。
3. 如果在 ELSyncing，则不会用过 L1 派生，关于同步模式可见：[ELSyncing Mode](./1_op_node_components.md#elsyncmode-and-clsyncmode)。
4. [AttributesHandler.Proceed](./1_op_node_components.md#proceed) 根据 Attributes 数据派生。
   1. 如果 PendingSafeL2Head < UnsafeL2Head，有两种可能一种是都正确就是落后了，正常更新 PendingSafeL2Head 根据新的 Attributes 数据。
   2. 如果发生分析，以 Attributes 为准，所以说 AttributesQueue 的数据源一定是正确的，详情见：[derivation data](./3_op_node_derivation_data.md)。
   3. 在派生的过程中如果发生了 BlockInsertPayloadErr 错误那就表述 L2Block 数据有问题，PendingSafeL2Head 需要回退到 SafeL2Head。
5. [Finalizer.PostProcessSafeL2](./1_op_node_components.md#postprocesssafel2)
6. [Finalizer.OnDerivationL1End](./1_op_node_components.md#onderivationl1end)
7. verifyNewL1Origin
8. 从 AttributesQueue 获取数据并设置给 [AttributesHandler](./1_op_node_components.md#attributeshandler) 等待下一次的派生。

**注意：[L1Traversal.AdvanceL1Block](./4_op_node_derivation_data.md#l1traversal) 在 EngineQueue.Step 之后会被调用，从 L1 拉取新的 block，为下一轮数据读取做准备。**

### Summary

EngineQueue 内部会维护一个 L1Origin 表明当前是在派生 L1Origin 上的 L2Blocks，L1Origin 来自 AttributesQueue 并且是距离最新块 15 个距离的 Block。

[AttributesHandler](./1_op_node_components.md#attributeshandler) 和 [Finalizer](./1_op_node_components.md#finalizer) 在 L1Origin 的上下文中派生其对用的 L2Blocks。
它们俩都是通过 [EngineController](./1_op_node_components.md#enginecontroller) 派生数据和设置 op-geth 的 [MetaData](./1_op_node_components.md#metadata)。
所以，[EngineController](./1_op_node_components.md#enginecontroller) 是比较基础的重要模块，它维护 op-geth 在 op-node 上的状态，并且对齐进行控制，例如：Finalizer 决定 FinalizedHead 通过 EngineController 设置给 op-egth；
AttributesHandler 对 LOrigin 上的 L2Blocks 数据进行派生，并角色 SafeHead 和 UnSafeHead，然后通过 EngineController 发送给 op-geht。

[AttributesHandler](./1_op_node_components.md#attributeshandler) ，根据 AttributesQueue 提供的数据驱动 EngineController 进行派生，在派生的过程中可能会处理 reorg 都会以 AttributesQueue 数据为准。
并且，在处理 op-geth 返回的 L2Block 数据错误（状态校验失败等）时会控制 L2 回滚（也是 reorg 的一种）到 SafeHead。

[Finalizer](./1_op_node_components.md#finalizer) 是一个比较有意思的模块，它用来决定 L2FinalizedBlock，这个策略的基本思路是：

* 记录每个 L1Origin 时刻对应的最大的 L2SafeBlock，保存129个版本（写死固定），然后在这129个版本中选择距离目前 L1FinalizedBlock 最近的那一份记录的 L2SafeBlock 作为当前 L2 的 L2SafeBlock。
* 在校验阶段会判断目前 L1FinalizedBlock 以及那份历史记录中的 L1Origin 是否都在 L1 上并校验其 Hash。

这个验证逻辑是一个典型的递归逻辑，L1FinalizedBlock 是目前获取的到的最新的数据，它如果在 L1Chain 说明他是安全的（理论上 Finalized 不能回滚，万一异常发生回滚也可以处理），
历史中距离 L1FinalizedBlock 最近的 L1Origin 如果也在 L1Chain，并且 L1Origin.Number <= L1FinalizedBlock.Number 说明历史中的这个 L1Origin 一定是安全的，
那么历史中 L1Origin 的这个时刻对应的 L2Safe 一定是安全的，否则就会被回滚掉或者被最新的历史覆盖掉（Finalizer 在插入历史的时候永远会用心来的覆盖旧的）。







