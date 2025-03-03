# op-node client

## L1Client

L1Client 是 op-node 访问 L1 的一个总封装，主要包括两个层次：
1. rpc 层：基于底层的裸 rpc 链接的封装，在这里有一个点需要注意如果 L1 的地址配置的是多个会在 rpc 层的基础上封装一层 fallback（后面介绍）。 
2. eth 语义层：基于 rpc 层实现的 eth 语义，在本层针对 GetReceipt 做了一个特殊的模块 CachingReceiptsProvider，因为 op-node 的派生逻辑以来 receipt，并且他的数据量相对较大一些，所以给他做了一个 cache，并且有特殊的逻辑（后面介绍）。

下面看一下，L1Client 在代码层面的层状结构。首先，底层的裸 client.RPC 或者是 fallbackClient（基于 client.RPC 增加了回退的封装），在此基础之上简单的封装了一层 InstrumentedRPC（这层主要是做的监控，没有特殊逻辑）。
然后，封装了 EthClient 层，这层封装了基本的 ETH 操作，更主要的是实现了一些 cache，其中一个重要的就是上面说的 CachingReceiptsProvider。
最后，在 EthClient 封装了 L1Client，他仅仅实现了四个接口，应该是 op-node 使用的，即 op-node 仅仅使用了四个 L1 的借口。

L1Client 仅有四个 API 供 op-node 使用，即 `L1BlockRefByLabel`，`L1BlockRefByNumber`，`L1BlockRefByHash`，`GoOrUpdatePreFetchReceipts`。
其中 `L1BlockRefByNumber`，`L1BlockRefByHash` 底层直接调用的 `eth_getBlockByNumber` 和 `eth_getBlockByHash` 底层的接口，没有什么特别的。其余两个值得讨论一下。

**注意：`L1BlockRefByLabel`，`L1BlockRefByNumber`，`L1BlockRefByHash` 在调用 eth json api 之前做了一层 headercall 的封装，他们底层返回的是 header，然后转换成 `L1BlockRef` ，其内部并没有交易只是 header 的部分信息。**

### L1BlockRefByLabel

底层都是调用的 `eth_getBlockByNumber` 方法，其中 blockNumber 参数用的是 `Label = Unsafe / Safe / Finalized`.
但是，Finalized 是比较特殊的，他会优先尝试使用 `eth_getFinalizedHeader` 方法，当 L1 不支持的时候会回退使用 `eth_getBlockByNumber`。

* `eth_getBlockByNumber`
  1. `Earliest(0)`：返回 genesis block。
  2. `Pending(-1)`：如果不是 miner 会回退到 Latest。
  3. `Latest(-2)`：返回 BlockChain 的 CurrentBlock，即最新 Inert 的 block。
  4. `Finalized(-3)`：最新被 fast finality 确认的 block。
  5. `Safe(-4)`：fast finality 计划确认的下一个 block，或者说正在确认的 block。

其中 Latest(-2，Unsafe)，Finalized(-3)，Safe(-4) op-node 会使用，其他的 label 目前没有被使用。

* `eth_getFinalizedHeader`
  1. 从 fast finality 计划确认的下一个 blcok 开始（`eth_getBlockByNumber` 的 `Safe(-4)`），往回 len(CurrentValidators) 个 Block（**这些block必须来自不同的 validator 否则就会返回 error**）。
  2. 这个结果与最新被 fast finality 确认的 blcok （`eth_getBlockByNumber` 的 `Finalized(-3)`）相比的大数值
  3. 所以，多数情况下与 （`eth_getBlockByNumber` 的 `Finalized(-3)`） 返回结果一致，仅仅在 fast finality 失效的时候才会起作用。

Finalized：op-node 会优先使用这个接口，不支持的情况下会使用上面的 `eth_getBlockByNumber` + `Finalized`。

**`eth_getFinalizedHeader` 接口的参数之前是固定的 21，后面优化为 -3 代表是全部的  len(CurrentValidators)，这里没有向前兼容，如果是其他数字会直接返回 error。**

### GoOrUpdatePreFetchReceipts

GoOrUpdatePreFetchReceipts 后台有一个写成会根据 block 进度自动并发的获取 receipt，理论上调用一次即可，op-node 只会在 reset 阶段会调用。
其参数就是 reset start block，所以每次会清理掉 CachingReceiptsProvider 的内容，可能会有 reorg 等因素（因为参数只有 block number，没有 block hash）并不会精确删除，而是全部删除。
另一个原因是，理论上这里缓存的 receipt 都是将来用于派生的，派生过的 receipt 理论上 op-node 不会访问，所以缓存也没有意义，所以既然有 reorg 的情况的发生，直接全部清除。

1. L1BlockRefByLabel 获取 Unsafe block，如果 currentL1Block > blockRef.Number sleep 3s 然后重来，否则 currentL1Block - blockRef.Number 区间就是 fetch 目标。
2. 如果 currentL1Block - blockRef.Number > s.maxConcurrentRequests / 2，配置的 rpc 请求的并发度（目前是 20），则会缩减。
3. 根据上面计算的并行度并发的获取 receipt。
   1. 使用 L1BlockRefByNumber 接口获取 block 信息（hash，number 等，并没有 txns）。
   2. 根据 block hash 确认 CachingReceiptsProvider 是否缓存了 receipt，如果存在则转位 step 4。
   3. 否则，调用 `eth_getBlockByHash` 方法，但是用 blockCall 进行了封装（会返回 block 信息而不是想前面一样返回 header 信息），获取 block 并写入 CachingReceiptsProvider。
   4. 如果 CachingReceiptsProvider 满了或者写入错误，会 sleep 1s，然后重来。
4. 在并发获取 receipt 的外面有一个管道接受获取的结果，判断 reorg。
   1. currentL1Block 的 Hash 是不是这一批 block 中最老的那个的 parent hash，如果不是并且 currentL1Block >= sequencerConfDepth(15 写死) + taskCount， 则 currentL1Block -= (sequencerConfDepth - taskCount)。
   2. 更新 currentL1Block，parentHash = latestBlockHash
   3. **注意：这面 reorg 的判断仅仅判断了批量获取 receipt 之间的连续性，并没有判断批量 receipt 内部的连续性，即并发之间的连续性，应该是考了到了性能和复杂度，并且上层的业务逻辑应景考虑这个问题了，所以这里没有处理。**


### FallbackClient

FallbackClient 是底层 rpc 级别的回退方案，在初始化阶段后有一个后台异步协程扫面最近 1 Minute 内的错误数量超过闸值 10 就会切换，每分钟这个错误数量会进行清零。
以下三类错误会被统计：
1. `ErrNoResult = errors.New("JSON-RPC response has no result"`
2. `NotFound = errors.New("not found")                          // NotFound is returned by API methods if the requested item does not exist.`
3. `rpc.Error interface                                         // Error() string and ErrorCode() int `

FallbackClient 针对是配置了多个 L1 Address 的时候会启动，如果只配置了一个地址则不会启动，**并且，第一个地址会被认为是最优先使用的，因此一旦不是第一个地址的话，后台还会尝试切换回第一个地址，知道成为为止。**
下面看一下切换或者说 Fallback 的核心逻辑：
1. currentIndex 记录了当前使用的地址索引，只会增加不会减少，所以所有的 rpc 轮训一次之后就结束了不会重新来。
2. currentIndex 仅仅在后台切换成第一个地址的时候会被重置 0，可以理解为永远在尝试使用第一个地址，后面的地址只会被轮训一次。
   1. 只有第一个地址可用了再出问题后面也会有机会在轮询到，如果第一个没有重链成功后面地址只会尝试一次。
3. 每次切换会 currentIndex++，如果不是第一个地址并且后台没有尝试重新链接第一个地址，会启动一个后台任务尝试链接第一个地址，一旦成功了会切换为第一个地址的链接。

## confDepth

confDepth 继承自 L1Client 并且只覆盖了 `L1BlockRefByNumber` 这一个方法，它的主要想法是获取到一个安全的 block，这个安全怎么定义的呢，就是距离最新块的距离，
定义一个安距离（目前的配置是 15）当需要的 block 在距离最新块的安全距离之前就可以直接调用 `L1Client.L1BlockRefByNumber`，否则返回 `NotFound`。
1. op-node 目前记录的最新块是空块（刚启动或者reorg），会直接获取 block 不做判断。
2. 如果访问的 block num == 0 || 安全距离 == 0 || 访问的 block num 大于安全距离，则可以获取 block 信息通过 L1Client。
3. 否则，返回 `NotFound`。

## L1OriginSelector

L1OriginSelector 基于 confDepth 进行的封装，**并且仅仅实现了 `FindL1Origin` 方法**，其目的是根据 L2Head 获取下一个 L2Block 对应的 L1Origin，用于派生的基准获取 deposit txns，所以这个逻辑很重要。

### FindL1Origin

在正式介绍 `FindL1Origin` 之前，先介绍一个变量 `MaxSequencerDrift` 它表示 L2Block 对应的 LBlock 的时间差最大值，目前理论上 1个 L1Block 对应 3个 L2Block，但是如果 L1 停止了或者 op-node 访问 L1 出了问题（这里还比较特殊访问 Block 失败不可以忍受，访问 receipt 可以，后面详细介绍），
1个 L1Block 可以对应更多的 L2Block，最多是多少呢？取决于这个变量，即 L2Head.Time - L1Origin.Time <= `MaxSequencerDrift`，否则 对应的 L1Origin 必须 +1。

具体步骤如下：
1. `currentOrigin = L1Client.L1BlockRefByHash(l2Head.L1Origin.Hash)`，获取当前 L2Head 对应的 L1Origin，如果返回 error，那么直接返回所以对于这个接口不容忍失败，后面的可以容忍失败。
2. `pastSeqDrift := l2Head.Time+los.cfg.BlockTime > currentOrigin.Time + MaxSequencerDrift`，判断下一个 L2Block 是否超时
   1. `MaxSequencerDrift` 在 Fjord 升级之后（Sep-24-2024 06:00 AM +UTC）写死是 1800，配置值失效。
   2. 即 L1 停止 30min 之后不能出块。
3. `nextOrigin = L1Client.L1BlockRefByNumber(currentOrigin.Number+1)`，获取下一个 nextL1Origin。
   1. 这里有一个细节，如果已经超时即`pastSeqDrift == true`，这个请求没有超时时间，会一直等到返回结果。
   2. 否则，超时时间是 100ms。
   3. 如果返回 error，`pastSeqDrift == true` 返回 error，否则返回 `currentOrigin`。
4. `L1Client.FetchReceipts(nextOrigin.Hash)` 获取 nextOrigin 的 receipt 并缓存成功
   1. 注意区分 `GoOrUpdatePreFetchReceipts` 是预先缓存，`FetchReceipts` 属于正式使用，先从 CachingReceiptsProvider 里读，没有在从 L1 获取并缓存上。
   2. `nextL2Head.Time(pre + 1s) > nextOrigin.Time && (pastSeqDrift || receiptsCached)` 返回 nextOrigin
      1. `nextL2Head.Time(pre + 1s) > nextOrigin.Time` 表示确实是跨越了下一个 epoch。
      2. `(pastSeqDrift || receiptsCached)` 前者表示强制条件触发了，后者表示条件成熟了否则回退上一个也能接受。
      3. 一般来说 `pastSeqDrift` 满足，`nextL2Head.Time(pre + 1s) > nextOrigin.Time` 必然满足。
   3. 否则，返回 currentOrigin

总结：
1. 超时（nextL2Block.Time 对应的 oldL1Origin 时间超过 1800s），必须返回 nextL1Origin，否则返回 error。
2. nextL2Block.Time 跨越到写一个 epoch（nextOrigin.Time），并且 receipt 成功缓存，则返回 nextL1Origin。
3. 否则，返回 currentOrigin。

## EngineClient(L2Client + EngineAPI)

EngineClient 实际上是 op-node 访问 op-geth 的总封装，它的封层于 L2Client 比较类似不在赘述。之所以叫 EngineClient 是因为出了常规的 L2Client 之外还提供了一套控制接口。
op-node 可以理解为控制层，op-geth 是执行层，[EngineAPI](./3_op_node_engine_api.md) 正是这套控制接口。本部分重点介绍 L2Client 部分，控制层见 [EngineAPI](./3_op_node_engine_api.md)。

L2Client 部分提供了 `L2BlockRefByLabel`，`L2BlockRefByNumber`，`L2BlockRefByHash`，`SystemConfigByL2Hash`，`OutputV0AtBlock`，五个接口。
1. `L2BlockRefByNumber`，`L2BlockRefByHash`：与 L1Client 类似根据参数返回信息，但是底层不再是 headercall 而是 payloadcall 返回的信息比较多包括 txns。
   1. L1Client 用 headercall 而不是 blockcall 的原因是 block 中大部分信息没有用，只需呀 receipt，所以 L1Client 的主要维护 blockinfo 和 receipt 并且分别维护。
2. `SystemConfigByL2Hash`：根据 hash 返回 payload，类似 `L2BlockRefByHash`，然后解析出 SystemConfig，BatcherAddr/Overhead/Scalar/GasLimit ，应该是每一个 L2block 的第一个 deposit tx 的内容。
3. `OutputV0AtBlock`：调用 op-geth 接口返回 L2ToL1MessagePasser 合约的 proof（只有一层 account 的 proof，这个接口只有对外的 api 使用。
4. `L2BlockRefByLabel` 
   1. `Pending(-1)`：miner.PendingBlock => blockchain.CurrentHeader。
   2. `Latest(-2)`：blockchain.CurrentBlock，状态更新后的最新 block。
   3. `Finalized(-3)`：blockchain.CurrentFinalBlock，来自 EngineAPI 的 forkchoiceUpdated 设置，即来自 op-node 的控制。
   4. `Safe(-4)`：blockchain.CurrentSafeBlock，来自 EngineAPI 的 forkchoiceUpdated 设置，即来自 op-node 的控制。

其中 `Latest(-2)`，`Finalized(-3)`,`Safe(-4)`，op-node 会用到。
