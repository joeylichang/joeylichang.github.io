# op-batcher data

## Overview

本部分与 [op-node.DerivationData](../op_node/4_op_node_derivation_data.md) 是对称的，本部分是提交 L2Block 到 L1Chain，
op-node.DerivationData 是从 L1Chain 中解析 L2Block。L1Chain 就是是网络一样，两边更像是网络编程，上链的 L2 数据可能有重复，
甚至有分叉的错误数据，但是**一定没有漏传**，两侧的代码就需要处理这些情况，好在封装的比较好，如果不想关心细节就只需要知道两次最顶层的
鞥装[AttributesQueue](../op_node/4_op_node_derivation_data.md#attributesqueue) 和 [ChannelManager](#channelmanager) 
总能按需拿到连续的数据，一共派生或者发送。

目前 opBNB 在硬分叉之后都适用的是 SpanBatch，这也是后面我们介绍的重点，这里又及点是很重要的，即使不关心细节也是要知道的：
* 针对 SpanBatch 的情况，channel 必须写满（压缩之后的数据大于设置的最大值），才能按 Frame 粒度往外读。
* 最后一个 Frame 会被设置为 isLastSpan，op-node 在派生的时候遇到这个标志位会使得相应的 L2Block 成为 L2SafeBlock。
* 所以，op-batcher 上传的数据以一个 channel 内的 L2Block 数量为粒度推进 op-node 侧的 L2SafeBlock 进度。
* 这个 L2SafeBlock 能分别回滚？

## ChannelOut

ChannelOut 是一个接口其两个实现是 SingularChannelOut 和 SpanChannelOut，在这里我们就介绍 SpanChannelOut，这也是目前在使用的。
SpanChannelOut 起内部设计三个基础的数据结构：SingularBatch，SpanBatch，RawSpanBatch。

一个 Block 对应一个 SingularBatch（去除了必要的数据，例如前面的 deposit tx），多个 SingularBatch 组成一个 SpanBatch
（内部一个 SingularBatch 生成的 SpanBatchElement 数组及操作相关的成员变量），一个 SpanBatch 对应一个 RawSpanBatch
（对数据做了进一步的精简）。

跟最初的 SingularBatch 相比设计的 SpanBatch 以及 RawSpanBatch 最主要的目的就是将多个 Block 的数据合并成一个大的密集的数据结构，增加压缩比。
RawSpanBatch 相当于将多个 Block 的数据合并成一个大的 Block 并且数据做了精简，然后进行压缩切分成 Frame 上传给 L1Chain。

SingularBatch 与 SpanBatch 最大的区别是，channel 必须写满之后才能压缩并且开始读取，都则读不出来数据（[ChannelBuilder](#channelbuilder) 对这点做了保障）。
SingularBatch 就是可以写点读一点，相对来说压缩比就没那么乐观和 SpanBatch 相比。

### SpanChannelOut

SpanChannelOut 正是针对 SpanBatch 的封装，他封装了数据的写入和输出，内部维护了上述三种基础数据结构的转换以及写满才能压缩的逻辑。
1. 数据入口：AddBlock 内部会调用 AddSingularBatch，完成上述三种结构的转换，如果写满会设置 FullErr。
2. 数据出口：
   1. OutputFrame 在数据写满的时候会根据设置的 MaxFrameSize（～120K）以 Frame 粒度读区数据。
   2. 如果 SpanChannelOut 被 Close（[ChannelBuilder](#channelbuilder) 层调用）会设置 Frame 的 isLastSpan 为 true。
3. 在实现上，每次写入 Block 都会生辰一个新的 RawSpanBatch 并进行 rlp 编码，如果超过设置大小就会压缩，直到压缩之后的大小超过配置值（～720K），会设置 FullErr。
   1. 这里每次写入都会编码压缩，使用了双 rlp buffer 减少 copy 大小和次数（因为编码之后大小不超过闸值说面白白copy了）。

## ChannelBuilder

ChannelBuilder 是在 ChannelOut 基础上管理其吐出来的 Frame，以及管道的超时（这个很重要）。从一下四个维度介绍 ChannelBuilder：

1. 数据的入口：AddBlock，内部调用 ChannelOut 的 AddSingularBatch，相当于将数据写入到 SpanBatch。
2. 数据出口：NextFrame，返回一个从 OutputFrames 驱动出来且缓存在 ChannelBuilder 的 frame 数组。
3. 数据驱动：OutputFrames，从 ChannelOut 读出 Frame 数据并缓存在本地 frame 数组内。
   1. 这个接口会做判断，如果 ChannelOut 没有满是不会读出来数据的。
   2. 如果 ChannelBuilder 有 FullError，会关闭 ChannelOut 在读出来所有的 frame 数据。
   3. ChannelOut 被关闭的时候，会进行压缩，无论是否达到配置的闸值。
4. 管道过期的逻辑（core）：
   1. 管道过期是为一个 ChannelBuilder 自己给自己设置 FullErr 的情况，其他都是因为 ChannelOut 满了才会设置 FullErr。
   2. FullErr 的设置很重要，他会决定当前 channel 不在写入数据，并把已经写入的数据压缩并切分成 frame。
   3. 管道过期的单位都是 L1Block 的数量，设置为过期的块高，有三种情况：
      1. 初始化阶段：initL1OriginBlockNum + MaxChannelDuration(32)，即最多存储 32 个 L1Block。
      2. 每次 AddBlock：L2Block.L1OriginNum + SeqWindowSize(14400) - SubSafetyMargin(30) ，距离插入的 LOrigin 14370个 L1Block。
      3. 每次发送成功之后：上链成功的 L1Block + ChannelTimeout(1200) - SubSafetyMargin(30)，上链接的 L1Block + 1170。
   
总结：
1. 三者取最小值，为管道的过期块高，正常情况就是一个 channel 处理 32个 L1Block
   1. 正常情况下一个 channel 处理 96 个 L2Block，最少 32个 L2Block，最多 32 * 1800（MaxSequencerDrift） = 57600 = 个 L2Block。
2. L2Block 创建的跨度，不能超过 14400 个 L1Block，即12小时。
   1. op-batcher 处理的数据的 L1Origin 超过目前 SeqWindowSize，则不会被提交成功。
   2. 结合 [BatchQueue.CheckBatch](../op_node/4_op_node_derivation_data.md#batchqueue) 可以知道，一个不会发送，一个不会解析，L2Chain 就会停止出块。
3. 全部 L2Block 数据上链的过程不能超过 1200 个 L1Block。

## channel

channel 在 ChannelBuilder 的基础上针对 tx 粒度对其携带的数据（Frame数组）进行管理。

1. 数据入口（AddBlock）和数据驱动（OutputFrames）都是透传给 ChannelBuilder。
2. 数据出口：NextTxData 需要说一下的是每次会吐出来 TargetNumFrames（6）个 Frame 数据组装成 BlobData。
3. 数据管理：既然要管理 tx 粒度的 frame 数据，那么 tx 直接成功失败需要周知 channel。/
   1. TxFailed，在 pendingTransactions（正在发送的数据）中删除，并将 Frame 数据写回 ChannelBuilder。
      1. 这个时候 Frtame 数据是乱序的，但是不要紧 [op-node.BatchQueue](../op_node/4_op_node_derivation_data.md#batchqueue) 在接收端会重新排序整理。
   2. TxConfirmed，在 pendingTransactions 中删除，添加到 confirmedTransactions
      1. 如果超时返回剩余的 blocks，供 ChannelManager 插入其他的 channel 中。
      2. 如果 Full 且都提交完毕，会 true 表示结束，否则否是 false 会继续。
4. channel 的超时有两个，一个是来自 ChannelBuilder，另一个是自身维护的上链 L1Block 区间不能超过 ChannelTimeout（1200个-1小时）。
   1. 只有在 TxConfirmed 内部会用，上层的 ChannelManager 还是会使用 ChannelBuilder 的 timeout。

## ChannelManager

ChannelManager 对多个 channel 组成 queue 进行管理，是数据层对外的总接口。

1. 数据入口：AddL2Block，将 L2Block 插入 blocks，但是在插入之前会比较 L2Block 的连续性，否则会返回 ErrReorg，[op batcher reset](./2_op_batcher_reset.md)会进行回滚。
2. 数据出口和驱动：TxData
   1. 先遍历 channelQueue 获取第一个有数据的 ch，并调用 channel.NextTxData 并返回。
   2. 否则，查看 currentChannel（写入的 ch 游标）是否满了，如果是创建新的 ch 并插入 queue。
   3. 遍历 blocks 并往 currentChannel 内写入数据，要么 block 洗完了，要么 channel 写满了。
   4. **CheckTimeout，检查 [ChannelBuilder](#channelbuilder) 是否过期，如果是的话后面不会写这个 ch 了。**
   5. currentChannel.OutputFrames 驱动 ChannelBuilder 生产数据。
   6. 尽量组装 6个 Frame Data 返回 txData（有可能数据本身就不足）。
3. 数据管理：
   1. TxFailed，找到对应的 ch 然后透传给 [channel.TxFailed](#channel)。
   2. TxConfirmed，找到对应的 ch 然后透传给 [channel.TxConfirmed](#channel)，如果 ch 过期会返回剩余的 block 并添加到 blocks。