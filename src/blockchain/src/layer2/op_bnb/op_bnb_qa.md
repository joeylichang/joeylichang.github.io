# opBNB Q&A

### Q1: L1 和 L2 的 safe，unsafe，finalize 分别代表什么含义？IsLastSpan 的含义是什么？

L1 详情可见 [LiClient.L1BlockRefByLabel](./op_node/1_op_node_clients.md#l1blockrefbylabel)
1. unsafe(-2)：返回 BlockChain 的 CurrentBlock，即最新的 L1InertBlock。
2. safe(-4)：在 BSC 中表示 fast finality 计划确认的下一个 Block，或者说正在确认的 Block。
3. finalize(-3)：
   1. 从最新块开始往回，找到 len(validators) 个不同 validator 的 Block。
   2. 最新被 fast finality 确认的 Block。
   3. 二者选择最新的 Block。

L2 的情况与 L1 不同，L1是靠出块进度和共识协议决定三个 Block 的进度，L2 的这三个 Block 完全是 op-node 来推断并下发给 op-geth。
所以，需要知道 L2 是怎么推断这三个 Block。
1. unsafe(-2)：派生或者出新块的最新块，op-node 下发给 op-geth.BlockChain.CurrentBlock。
2. safe(-4)：op-node 收到 L1 的 Frame 数据回组成管道，管道的最后一个 Frame 会被设置为 IsLastSpan，对应派生出来的 L2Block 会被推断为 L2SafeBlock。
3. finalize(-3)：[Finalizer](./op_node/2_op_node_components.md#finalizer) 模块根据历史 SafeBlock 进行推断。
   确认的方法：当前正在解析的 L1Block 与其完成之后的 L2SafeBlock 会记录一个历史，总共会保存 129 个历史版本，用最新的 L1FinalizedBlock 在历史中找到距离其最近的 L1Block 的那条记录的 L2SafeBlock 作为当前的 L2FinalizedBlock。

IsLastSpan 表示 op-node 当前的接受管道（op-batcher 的发送管道是对称的，所以情况一样）最后一帧会被设置为 true。
在派生的过程中遇到被设置为 IsLastSpan 的 L2Block 会被设置为 SafeBlock，后面还会详细介绍在 [L1/L2 Reorg](#q10-l1l2-stop-或者-reorgop-node--op-batcher-如何处理)。

### Q2: op-node 和 op-batcher 常用配置项的含义都是什么?

op-node:
1. SeqWindowSize：当前正在解析的 L1DeriveBlock 与正在派生的 L2Block.L1origin 差距超过这个配置值就会停止解析数据，即 L2Block 所属的 L1Epoch 与上传 L2Block 到的那个 L1Block 差距过大，L2就会停止（上传和下载解析都会失败）。
2. ChannelTimeout：op-node 侧 Channel 的 L1DeriveBlock 之间的距离最大值，目前是 1200，即 1小时，否则拒绝接受。
3. MaxSequencerDrift：配置值失效，在 Fjord 升级之后写死 1800s，即目前一个 L1Block 最多可以对应 1800个 L2Block，换句话说就是 L1 停止了 L2 还能维持 30 min。 
4. driver.Config.VerifierConfDepth：派生的时候，获取 L1Block 距离最新 L1UnSafeBlock 的安全距离，目前配置是 15 针对 BSC 可以认为几乎没有 L1Reorg。
5. driver.Config.SequencerConfDepth：同上，但是是针对出块的时候。
6. driver.Config.SequencerMaxSafeLag：弃用，使用 MaxSequencerDrift。
7. driver.Config.SequencerPriority：在主循环会优先走出块的逻辑，在进行派生。
8. driver.Config.SequencerCombinedEngine：开启之后，在出块阶段会讲 GetPayload + NewPayload + ForkchoiceUpdate 的逻辑结合在一起合并成一个 RPC 请求发送给 op-geth

op-batcher:
1. MaxChannelDuration：一个管道的 L1BlockNumber 跨度，这个不是唯一决定管道存储的跨度，详情见 [ChannelBuilder](./op_batcher/1_op_batcher_data.md#channelbuilder)。
2. TargetNumFrames：一个 TX 写到 Blob 的个数，也是一个管道内生成的 Frame 个数，目前是 6 个。
3. MaxFrameSize：一个 Frame的最大大小，目前大约 120K。
4. CompressorConfig.TargetOutputSize = TargetNumFrames * MaxFrameSize：压缩输出的目标大小，达到这个数据就会设置 isFull，管道在被使用后就会被关闭。
5. MaxPendingTransactions：txqueue 中最多在并发处理的 tx 数量，查过这个闸值后面来的 tx 就会排队等待。

flag:
1. --l1.trustrpc：标识信任 L1RPC 节点的数据，拿过来不会校验，如果 geth 和 op-geth 版本不一样字段有变化，BlockHash 如果重新计算就会失败，一般来说 op-geth 的版本是很难和 geth 版本一直的所以开启是常态。
2. --syncmode：同步模式，分为 execution-layer 和 consensus-layer。
3. --skip_sync_start_check：在 Reset（例如重启）会跳过 L2UnSafeBlock -> L2SafeBlock 的校验，直接开启 L2SafeBlock -> L2FinalizedBlock de 校验，多数情况 L2FinalizedBlock == L2SafeBlock，所以会提速，详情[op-node.Reset.FindL2Heads](./op_node/7_op_node_reset.md#findl2heads)。
4. --el_trigger_gap：落后多少块会自动触发 ELSync，默认 86400，即24小时。

### Q3: SeqWindowSize，ChannelTimeout，MaxChannelDuration，MaxFrameSize，TargetNumFrames 这几组配置深刻含义，容易混淆？

SeqWindowSize 主要描述的是 L1Origin 与 L1DeriveBlock（L2Block 上链的 L1Block）之间的距离不能超过配置值，即 L2Block 从生产出来到上链不能过这么大，目前配置 14400，即 24小时，否则数据集不发送也不解析。

ChannelTimeout 主要描述的是数据的发送（op-batcher）和解析（op-node）用到的 Channel 跨度不能超过的 L1Blocks 数量，值得是 L1DeriveBlock 跨度，即一个 Channel 上链不能超过这么大，默认是 1200 ，即 1小时。
op-batcher 与 op-node 会使用相同数值，因为 op-batcher 的数值是从 op-node 拉取过来的。

MaxChannelDuration 也是配置的 Channel 跨度，而且从起始的 L1BlockNumber 开始计算跨度选取最小的，理论上 MaxChannelDuration 会小于 ChannelTimeout，op-node 只会使用 ChannelTimeout。

MaxFrameSize，TargetNumFrames 分别表示一个 Frame 的大小和一个 TX 携带的 Frame 个数。目前 Frame 与 Blob 是一对一的关系，
即目前一个 TX 交易写到 6个 Blob，每个 Blob 是 ～120K（有一些边界元数据）的压缩之后的数据，一个 TX ～720K 压缩之后的数据。
目前配置的压缩率是 0.4（线上监控看 0.2-0.4之间）所以一个 TX 大概的原始数据大小 1.7M-3.5M，目前线上平均大小 21K，
所以目前一个 TX 包涵的 Block 数据大约在 85-171 ，但是 MaxChannelDuration 配置的是 32，对应 96 个 L2Block，所以一个 TX 的范围
大约是 85-92 之间。

### Q4: L2Block / Batch(SingularBatch/SpanBatch/RawSpanBatch) / Frame / Channel 之间的关系是什么？

L2Block 去除 L1 的交易（Deposit）之后是 Batch 数据，主要是去除 L1 层的荣誉信息，因为这个可以从 L1 上派生出来，没必要浪费空间。

Batch 是一个接口，目前使用的都是 SpanBatch，SpanBatch 是多个 SingularBatch，一个 Block 一个 SingularBatch，所以 SpanBatch
是对多个 Block 的组合增加压缩率节省 L1 空间，SpanBatch 在压缩之前会转换成 RawSpanBatch ，数据进一步精简（简单理解就是组装成了一个 Block，但是加了一些元数据又可以辅助其剥离成多个 Block）。

RawSpanBatch 压缩之后的数据会被切分成 Frame，切分根据配置的 MaxFrameSize 进行切分，最后一个 Frame 会被设置为 IsLastSpan。

Channel 是管理 Frame 的工具，上层通过 Channel 获取 Frame 进行发送，最后一个 Frame 生成之后 Channel 会被标记为 IsFull，
Frame 数据输出完会不能再被使用，op-node 接收到 IsLastSpan 的 Frame 之后会把对应的 L2Block 设置为 L2SafeBlock。

### Q5: op-node 怎么处理 op-batcher 提交的乱序数据？

[op-node.BatchQueue](./op_node/4_op_node_derivation_data.md#batchqueue) 这部分做了冗余数据兼容。

### Q6: op-node，op-batcher 调用了哪些 L1/L2 的接口？

[L1Client](./op_node/1_op_node_clients.md#l1client)
[L2Client](./op_node/1_op_node_clients.md#engineclientl2client--engineapi)
[L2EngineAPI](./op_node/3_op_node_engine_api.md)

### Q7: 客户端的 Fallback 逻辑是什么样的？

[FallbackClient](./op_node/1_op_node_clients.md#fallbackclient)

### Q8: 在 FullNode 模式下，CL 与 EL 是如何配合完成工作的？有什么配置控制？

[ELSync and CLSync](./op_node/2_op_node_components.md#elsyncmode-and-clsyncmode)

1. --syncmode：同步模式，分为 execution-layer 和 consensus-layer。 
2. --skip_sync_start_check

在 EL 模式下重启的时候会进行 [L2Rest](./op_node/7_op_node_reset.md)，一般会开启 --skip_sync_start_check。
多数情况下会尝试 --syncmode=execution-layer，如果失败会换成 --syncmode=consensus-layer，只要 L1Addr 没有问题就会派生但是可能会慢一点。

### Q9: L2 出块时间必须小于 L1 的出块时间吗？出块的时候是怎么选取的 L1Block？

[FindL1Origin](./op_node/1_op_node_clients.md#findl1origin) 描述了如何选择 L1Origin。

根据上述描述的逻辑，选择 L1Origin 的时候不是当前的就是下一个 L1Block，所以必须是连续的，当 L2 出块时间大于 L1 的时候是保证不了这点的。
当 L2 出块时间大于 L1 的时候，L1 上面的消息就会遗漏，即使不遗漏（取决于选择的算法）L2 远远落后 L1 的话，随着时间的流逝，L2 永远接不到 L1 的消息。

### Q10: L1/L2 stop 或者 reorg，op-node / op-batcher 如何处理？

* L1 Stop
   
   L1 停止的话，L2 仍然可以出块，但是最多出 1800 个 L2Block，即 30min，之后 L2 也会停止，这部分的逻辑在选择 [FindL1Origin](./op_node/1_op_node_clients.md#findl1origin)。
   如果 L1 在 30min 之后起来了，会发生什么？op-node 会退出，因为主循环中 Sequencer 只会处理 ResetErr，其他 err 直接跳出主循环，结束程序。


* L2 Stop

   SeqWindowSize 就是给这中场景设置的，简单说停止的时间少于这个配置的 L1Block 跨度，还会继续出块，但是会出空块（没有 L2 交易，只有来自 L1 的消息）， 
   因为 [Sequencer.PlanNextSequencerAction](./op_node/6_op_node_sequencer.md#plannextsequenceraction) 在计算下一次动作的时间，
   在出块的时候里面会比较 nextL2BlockTime 与 当前时间差值，如果为负，当前的出块时间就是 0。
   
   如果 L2 超过了 SeqWindowSize 之后重启呢，op-bathcer 不会提交，op-node 不会解析，相当于无法恢复。
   

* L1 Reorg
   
   op-bathcer 对于 L1 Reorg 没有逻辑上的兼容，即 L1 如果发生了 reorg L2 数据丢失，其实 op-bathcer 是不知道的，也没有补救措施。
   目前 op-bathcer 做的唯一保障就是 tx 上链确认高度距离最新块的距离，即 tx 所在的快搞距离最新块 N 个块被认为不会回滚，目前的使用的
   是默认配置 N = 10，可以通过 `--um-confirmations` flag 进行设置。

   op-node 需要监控 L1Chian 并从中连续没有遗漏的获取数据，所以对于 L1 Reorg 是有处理措施的。当派生逻辑中的底层模块
   [L1Traversal.AdvanceL1block](./op_node/4_op_node_derivation_data.md#advancel1block) 按顺序解析一个 L1Block 检验失败（L1 Reorg 可能是）
   会执行 [op-node.Reset](./op_node/7_op_node_reset.md) 回退到一个可靠的点，继续执行（在回退和重新执行期间，L1 确认了最长链，否则会往复这个过程）。

* L2 Reorg

   L2 的控制都在 op-node 所以发生了 reorg 也是 op-node 控制的，我们需要看一下在什么情况下发生 L2 Reorg，然后再看处理的措施。
   首先，op-node 控制的 L2 Reorg 只有两种情况下：
   1. [AttributesHandler.Proceed](./op_node/2_op_node_components.md#proceed) 在更新元数据的时候，如果 EngineAPI 返回 BlockInsertPayloadErr 的时候会设置 [EngineController](./op_node/2_op_node_components.md#enginecontroller) 的 Reorg 标志位。
      1. BlockInsertPayloadErr 主要是 op-geth 执行期间校验失败，比如 stateroot，blockhash 等。
   2. ConfirmPayload 返回来的 Block 比本地的 unsafeblock 小。

   接下来看一下 op-node 是如何处理的，op-node 对于上述两种情况都不会返回给主循环 err，而是 EngineController 内部记录下来了并后续处理，这样就不会整体的 op-node.Reset（op-node 解析的 L1Block 就不会回滚），op-node 就会继续在 L1Chain 上解析，这样后面正确的数据来了就会处理了 L2 Reorg。

   最后看一下 op-batcher 是符合处理，在 [AhannelManager.AddL2Block](./op_batcher/1_op_batcher_data.md#channelmanager) 的时候，因为 Block 不连续会清楚所有未上传的数据，然后进行 [op-bathcer.Reset](./op_batcher/2_op_batcher_reset.md)，他会向 op-node 获取安全的可靠点继续拉取 L2 数据并发送，回退并继续的这段时间，op-node 会处理完L2 Reorg。
   
   op-node 具备处理 op-batcher 提交的回滚数据，op-batcher 也尽量不上传回滚的数据，那么什么情况下会上传回滚的数据呢？管道未满之前的回滚数据不会发送，否则也会发送回滚的数据。

   最后的最后，还有一个话题，IsLastSpan 是 op-node 设置 L2SafeBlock 的重要标志，他是表示管道的最后一个 Block 数据，只能说一个完整的 SpanBatch 接收并处理完了没有丢失，但是并不是说一点是 Safe。
   因为可能整个管道的数据都会被回滚，所以 op-node.Reset 中 L2FinalizedBlock 才是最安全的点，虽然多数时候 L2FinalizedBlock = L2SafeBlock(这是系统运行事实，不是理论必然，但是理论上 Finalized 的产生一定是安全的，所事实上多数情况 Safe 也是安全的)。
   所以，IsLastSpan 怎么理解？就是表示一个 Channel 的数据完整被接受了而且问题，可以认为数据完整性上是安全的，但是从 Chain 的角度未必是最长链，把它设置为 SafeBlock，但是 Chain 中的 Safe 与 Finalized 还是有区别的，Finalized 才是真正的不会被回滚的。
   Safe 只能说明上 L1 了，不代表不能被回滚。

### Q11: op-node 中派生和出块这两个逻辑是如何协同完成工作的？

[op-node derivation and sequencer](./op_node/op_node.md#协同工作)

