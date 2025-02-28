# op-node

## Overview 

op-node 是整个 op 系统中非常核心的一个模块，它是整个系统的控制层，控制 op-geth 的执行。
op-geth 的元数据（UnSafe/Safe/Finalized） 都是 op-node 下发，不会自己更新，今儿也就控制了 op-geth 的数据流。

这部分要想理解好，主要分以下几个部分：
1. 基础层：[Client](./1_op_node_clients.md) 介绍了 op-node 使用了哪些 L1 和 L2 接口，并在基础之上对客户端做了哪些包装。
   1. [FallbackClient](./1_op_node_clients.md#fallbackclient) 介绍了 L1Client 回退的策略与实现。
   2. [confDepth](./1_op_node_clients.md#confdepth) 介绍了获取 L1Block 的安全距离的客户端。
   3. [L1OriginSelector](./1_op_node_clients.md#l1originselector) 介绍了如何牟定 L1Block 去定序 L2Block。
   4. [EngineAPI](./3_op_node_engine_api.md) 介绍了 op-node 控制 opgeth 的 api。
2. 组件层：[Components](./2_op_node_components.md) 介绍了基于 Client 之上的一些功能组件，他们组合在一起实现了派生和定序。
   1. [EngineController](./2_op_node_components.md#enginecontroller)  维护 op-gteh 元数据，与 op-geth 进行控制流交互，是派生和定序的底层基石。
   2. [Attributeshandler](./2_op_node_components.md#attributeshandler) 驱动派生逻辑。
   3. [Finalizer](./2_op_node_components.md#finalizer) 确定 L2FinalizedBlock。
   4. [Derivation Data](./4_op_node_derivation_data.md) 介绍了从 L1Block 的 receipt 数据中解析 L2Block 的全部组件。
3. 核心层：[DerivationPipeline](./5_op_node_derivation_pipeline.md) 和 [Sequencer](./6_op_node_sequencer.md) 介绍了 op-node 两个核心功能派生和定序。

除了上述内容外，还有四部分对于在更高层次上理解 op-node 也是很重要的：
1. [EngineController.MetaData](./2_op_node_components.md#metadata) EngineController 维护的元数据，可以通过对他们的梳理，理解派生和定序是维护哪些数据，怎么推进，怎么协同工作的。
2. [ELSync and CLSync](./2_op_node_components.md#elsyncmode-and-clsyncmode) 两种同步模式的学习，对于 P2P 和控制流会有一个更深刻的理解。
3. [DerivationPipeline and Sequencer](#协同工作) 从更高层面上整体的理解 op-node 在定序器模式下的工作流程。
4. [op node rest](./7_op_node_reset.md) 介绍了异常情况下 rest op-node，包括 reorg（重点）。

## EventLoop

op-node 的大循环中内容比较多，但是核心部分却不是很多也不是很难理解，这里面值介绍了一些用得到的事件处理。

1. sequencerCh: [Sequencer](./6_op_node_sequencer.md) 进行控制出新块。
   1. [RunNextSequencerAction](./6_op_node_sequencer.md#runnextsequenceraction) 出最新的 block。
   2. [PlanNextSequencerAction](./6_op_node_sequencer.md#plannextsequenceraction) 计算下一个步骤需要的时间，然后等待上一个时间的结束，然后重置定时器，然后再下一轮的 RunNextSequencerAction，这样保证了上一一轮的充足时间。
2. stepReqCh: [DerivationPipeline](./5_op_node_derivation_pipeline.md) 进行派生
   1. 如果是 ELSyncMode，则直接返回。
   2. syncStep 
      1. [DerivationPipeline.Step](./5_op_node_derivation_pipeline.md#enginequeuestep)，开始派生数据
   3. err == ErrCritical，return
   4. err == EngineELSyncing，continue
   5. err == ErrReset, DerivationPipeline.Reset() => 元数据全部规正一下到安全位置
   6. err == ErrTemporary/NotEnoughData/others，随机避让时间，然后进入 stepReqCh 进行下一轮派生
3. l1HeadSig
   1. 传递给 l1State
   2. 随机避让时间，然后进入 stepReqCh 进行派生
4. l1SafeSig
   1. 传递给 l1State
5. l1FinalizedSig
   1. 传递给 l1State
   2. 更新 [Finalized](./2_op_node_components.md#finalize) 并尝试更新 L2SafeBlock。
   3. 随机避让时间，然后进入 stepReqCh 进行派生
6. unsafeL2Payloads，来自 P2P 的节点
   1. 如果是 CLSyncMode 或者 ELSync 结束了
      1. [ClSync.AddUnsafePayload](./2_op_node_components.md#clsync)，待 stepReqCh 的 syncStep 处理。
      2. 随机避让时间，然后进入 stepReqCh 进行派生
   2. 如果是 ELSyncMode
      1. 如果 ref.Number <= s.engineController.UnsafeL2Head().Number，continue
      2. [EngineController.InsertUnsafePayload](./2_op_node_components.md#insertunsafepayload) 出入数据。

注意：
1. l1HeadSig 和 l1FinalizedSig 都需要进入派生阶段，而 l1SafeSig 不需要。
2. ELSyncMode 是不需要派生的，也不需要 Seuqnecer 出新块（[StartPayload](./2_op_node_components.md#startpayload) 有判断）。

## DerivationPipeline && Sequencer

op-node 最重要的两个核心功能：派生和定序（出新块）。二者在工作流程中有很多重合，如何配合工作，从代码级别能够玻璃清晰对于我们理解 op-node 的核心直观重要，这也是本部分的重点。
下面从几个维度来看一下二者的区别与关系。

### 功能
* DerivationPipeline 是负责处理来自 L1Block 的数据，通过 receipt 解析出 L2Block，通过 EngineController 发送给 op-geth。
* Sequencer 根据 L1Block 的信息生成 PayloadAttribute 模板，通过 EngineController 发送给 op-geth 让其往里打包交易。

### 工作模式
* FullNode：只有 DerivationPipeline 即可，通过参数配置可以让 op-node 从 L1 解析数据发送给 op-geth 或者 op-egth 自己通过 P2P 同步数据，到了最新块之后转成从 L1 同步即可。
* Sequencer：对于开启了定序功能的节点（目前全局唯一），需要出块所以需要开启 Sequencer 模块，同时也会开始 DerivationPipeline 模块，因为它在这里一个重要的作用是判断 L2 的数据是否上链 L1（抗审查），并且根据解析的数据判定 L2SafeBlock 和 L2FinalizedBlock。

那么在 Sequencer 模式下两个模块是如何配合工作片的呢？就显得尤为重要。

### 协同工作

首先，Sequencer 会根据 EngineController.unsafeHead 进行推进生成新的区块，DerivationPipeline 根据 EngineController.pendingSafeHead 推进派生的进度。
这里说一下，op-batcher 会定时的向 L1 发送数据，派生解析的数据是 L1最新块之后 15 个块的数据，这已经是 safe 数据了，但是对于 L2 来说不一定因为可能提交了一些 fork 数据，
所以此时的 safehead 只能是 pending，需要进一步的确认信息 IsLastSpan(op-bathcer 部分会介绍)。DerivationPipeline 还需要一定的手段判定 EngineController.finalizedHead，
详情可见[DerivationPipeline Finalizer](./5_op_node_derivation_pipeline.md#summary)。

其次，就是串行与 op-getj 交互，DerivationPipeline 和 Sequencer 都会调用 EngineController 向 op-geth 发送数据，这里 [op-node 总循环](#eventloop) 影响都是穿行没有并行，并且 EngineAPI 对于重复数据有一定的兼容性。

