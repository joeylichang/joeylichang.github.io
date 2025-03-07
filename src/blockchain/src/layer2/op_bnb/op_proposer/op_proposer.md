# op-proposer

op-proposer 的逻辑相对简单一些，就是获取 op-geth 上 L2ToL1MessagePasser 合约的提款 proof（StateRoot -> StorageRoot 一级 trie 的 proof） 
提交到 L1 的 L2OutputOracle 合约上，用于提款时进行验证。

1. 每间隔 6s 执行一次主循环的内容。
2. 向 L1 的 L2OutputOracle 合约发送请求，获取她需要的下一个状态的 BlockNumber。
3. **向 op-node 请求目前的状态，获取到当前的进度信息（finalized ，如果设置了 AllowNonFinalized ，可以用 safe 作为当前进度信息）。**
4. 比较从 L1.L2OutputOracle.NextBlock 是否领先进度信息，否则返回 error。
5. 等待（6s 检查一次） txmgr 看到的 BlockNumber 大于 op-node 返回的 L1 进度的 L1UnSafe + 1，之后才能继续。
6. **向 op-node 发送 OutPutAt 请求获取 proof，op-node 会转发给 op-geth。**
7. 发送结果给 L1 的 L2OutputOracle 合约。