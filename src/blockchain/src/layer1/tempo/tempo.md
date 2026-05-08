# Tempo

Tempo 是一条"支付优先（payments-first）"的 EVM 兼容 L1 区块链，由 Stripe 与 Paradigm 孵化。
针对支付场景做了深度优化的通用区块链，目标是提供**即时确定性结算（instant deterministic settlement）**、**可预测的低费率**、以及**稳定币原生的使用体验**。它不是"顺便支持支付的 DeFi 链"，而是"也支持通用智能合约可编程性的支付链"。

## 区块交易结构
```
┌─────────────────────────────────────────────────────────┐
│  Block N (500M total, 500ms)                             │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ ① NonShared (≤ 450M) ── Proposer 自己的池             │  │
│ │   ├─ general 类硬上限 30M（DeFi、合约调用）             │  │
│ │   └─ payment 类用剩下的 ≥420M（TIP-20 转账）            │  │
│ ├─────────────────────────────────────────────────────┤  │
│ │ ② SubBlocks (≤ 50M) ── 每 validator 50M/N 配额         │  │
│ │   ├─ V₁ 的 subblock：≤ 50M/N                           │  │
│ │   ├─ V₂ 的 subblock：≤ 50M/N                           │  │
│ │   └─ ...（按 metadata 顺序串行执行）                    │  │
│ ├─────────────────────────────────────────────────────┤  │
│ │ ③ GasIncentive ── unused subblock 容量回归 proposer    │  │
│ │   = ∑(50M/N − Vᵢ_used)                                │  │
│ │   proposer 在这段塞之前因 ① 30M cap 挤掉的 general tx  │  │
│ ├─────────────────────────────────────────────────────┤  │
│ │ ④ System ── 尾段系统交易                                │  │
│ │   含 subblocks_signatures_tx（subblock 签名校验） 等    │  │
│ └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

* 500ms 出块出块即 finlized(Simplex BFT)，500M GasLimit。

    TIP-20 precompile 调用（**原生 Rust，不走 EVM 解释器**）。TIP-20 精确成本：
    * 一笔已存在收件人的 TIP-20 转账 = 50,000 gas = $0.001
    * 一笔到新地址的 TIP-20 转账 = 300,000 gas（含 250K 新账户成本）

    调一次 TIP20Token::transfer precompile，几个 storage SLOAD/SSTORE + 几次算术，纳秒级到微秒级。420M / 50K = 8400 笔 TIP-20 转账（payment lane 满载场景），Reth 16K TPS 意味着 500ms 内理论能跑 8000 笔（EVM），加上原生 precompile 比 EVM 字节码再快 1-2 个数量级。


* SubBlock TX
    * SubBlock TX Type
        1. 必须是 AA 交易（TempoTransaction，type 0x76） —— 而不是 Legacy / EIP-1559 / EIP-2930 / EIP-7702
        2. 它的 nonce_key 第 1 字节必须是 magic prefix，紧跟 15 字节是发送 subblock 的 validator 的 partial pubkey
        5. 可以是（AA (0x76) + subblock-targeted nonce_key 交易）：
            1. TIP-20 转账（最常见，因为支付场景最多）
            2. DEX swap（用户怕被 MEV 抢跑，把 swap 路由到特定 validator）
            3. 批量调用（AA 原生支持，一笔交易调多个合约）
            4. 任意合约调用（治理投票、NFT 铸造、合约部署）
            5. Fee sponsorship 调用（应用替用户付 gas）
        6. 不可以是：
            1. Legacy，EIP-2930 (access list)，EIP-1559，EIP-7702 (delegation)——没有 nonce_key
            2. AA (0x76) + 普通 nonce_key —— nonce_key 没编码 validator

    * SubBlock TX Constraint
        1. 单 validator 配额限制 = 50M / N，如果 N = 10 个 validator，每人 5M gas，50K gas 的 TIP-20 转账：每个 subblock 能塞 100 笔，5M gas 的复杂 DeFi 交易：一个 subblock 最多塞 1 笔（且可能超额），**subblock 容量有限，validator 自己会挑高频小笔的优先（费用密度高），低频大笔的可能被挤出**。
        2. subblock 内 tx 必须能在 parent 状态上独立执行。依赖未确认状态的 tx 进不来。比如 "用户 A 先调 transferFrom 拿了授权额度、再调 swap" 这种链式依赖，必须打包成单笔 AA 多 call 才能进 subblock，分两笔的话第二笔会被拒。
        3. 必须是用户主动构造（钱包/SDK 默认不会触发）。主动构造 AA 交易，主动设置 nonce_key 前缀 + 编码目标 validator，用户显式选择目标 validator——需要 Tempo 生态里的专用钱包/SDK 提供这个选项。

    * TX Packing Order
        1. 0-200ms: 执行 NonShared 段 tx; 并行收集 subblock
        2. 200-450ms: 执行 SubBlock -> GasIncentive -> System + 计算 state root。
        3. 450-500ms: Broadcast block 提议，收到 ≥2/3 投票。

    * TX conflict
        * SubBlock 与 NonShared 交易冲突
            1. proposer 直接出一个只有 NonShared + System 段的纯净块。
            2. roposer 不能"挑选"subblock 里能成功的 tx 而丢弃失败的——要么全包，要么不包，防止 proposer 通过"过滤 subblock tx"做隐性审查。
        * SubBlock 之间的交易冲突
            1. 正常包进，failed receipt，gas 被收，value 不动。


**支付 SLA(Payment Lane)、AA 交易抗审查（SubBlock）、容量利用率（GasIncentive）**

## TIP-20(稳定币和支付代币原生代币标准)

Gas fee 必须是 currency = "USD" 的 TIP-20 token——具体是 PATH_USD / USDC / USDT 还是别的 USD 稳定币都行，Fee AMM 自动兑换给验证者。EURC 等非 USD 稳定币目前主网不支持付 gas（属于 Onchain FX 未来路线图）。

> PATH_USD 是 Tempo 协议层的"默认 USD 稳定币"——预编译部署在地址 0x20C0...000，扮演整个经济系统的"参考美元"和"流动性中枢"。协议指定的 ISSUER_ROLE 才可以 mint，储备背书来自 Tempo 合作的银行/机构。

### TIP-20 体系
1. 基础层: 兼容 ERC-20(transfer/approve/allowance/balanceOf/totalSupply/name/symbol)。
2. 对账层：xxxWithMemo 允许用户转账有附言，银行/企业对账场景。
3. 货币声明层：ISO 4217 currency identifier (不可变) ，对于 ERC-20 链上无法知道 token 是 USD/EUR，DEX 无法路由。
4. 合规层：TIP-403 policy/RBAC/burnBlocked/supplyCap/pause/unpause，对于 ERC-20 每发行方各自实现黑白名单，跨发行方合规工具无法做。
5. Yield 层：opt-in reward distribution，对于 ERC-20 计息稳定币要靠链下计算，差/延迟/不可审计。
6. Tempo 集成层：DEX 报价币/Virtual Addresses/Fee Token，ERC-20 跟链的支付/DEX 解耦，协议没法做支付优化。

#### 对账层

TIP-20 原生支持 `transferWithMemo(to, amount, bytes32 memo)`。直接对标银行 ISO 20022 标准。Memo 承载了后端系统自动对账（无需人工介入）所需的支付上下文。

典型的 memo 内容：发票编号、客户编号、成本中心代码、订单号、PO（采购订单）号。这与 SWIFT 报文的工作方式一致——每一笔支付都携带一段结构化引用，下游 ERP 系统可以自动解析。

| 数据大小 | 处理方式 | 链上内容 |
|---|---|---|
| ≤ 32 字节 | 直接 memo 字段 | 完整引用数据（发票 ID、客户 ID 等） |
| > 32 字节 或 含 PII | Commitment（承诺）模式 | 仅存哈希或定位符 —— 完整数据保存在链下 |

#### 货币声明层

ERC-20 现实：合约只有个 symbol = "USDC" 字符串，字符串自由填。攻击者可以发个山寨币 symbol 也叫 "USDC"。链上没有任何机制告诉你"这个 token 是美元、欧元还是日元"。

TIP-20 解决：每个 token 在部署时烧入 ISO 4217 货币代码（"USD"、"EUR"、"GBP"、"SGD"、"JPY"...），之后永远不能改。

#### 合规层

TIP-20 合规四件套：
1. TIP-403 Policy Registry：把合规策略**外置到一个共享注册表**——所有 TIP-20 token 引用同一份策略
2. RBAC 5 角色(TIP-20 内置)：内置 PAUSE / UNPAUSE / ISSUER / BURN_BLOCKED / DEFAULT_ADMIN，标准化合规操作
3. **supplyCap + pause**：协议级供应控制 + 紧急暂停
4. TIP-1006（burnAt / burnBlocked）：合规团队对受制裁地址执行强制销毁， in review

##### TIP-403

ERC-20 合规痛点: N 个发行方 × M 个 token 个独立的黑名单合约——没有统一接口，每家自己实现，bug 风险各自承担，并且策略粒度不够精细（sender 和 recveiver 制裁名单不一致时，需要复杂合约逻辑）。

1. TIP-20 中可以设置 `transfer_policy_id`.
2. TIP-403 支持三种策略：WHITELIST，BLACKLIST，COMPOUND（TIP-1015 引入组合策略）
    1. WHITELIST：只有 KYC 认证地址可以使用，适合：机构稳定币、tokenized deposit、监管证券。
    2. BLACKLIST：OFAC 制裁地址不可使用，适合：USDC、USDT 等大众消费稳定币。
    3. COMPOUND：让 sender / recipient / mintRecipient 各用一种策略。
3. TIP-403 规定四种角色：
    1. Transfer：sender + recipient 都查
    2. Sender： 仅查 sender 
    3. Recipient： 仅查 recipient
    4. MintRecipient：仅查 mint 接收方
4. TIP-20 调用 TIP-403
    1. transferPolicyId 获取策略 id
    2. 调 TIP-403 isAuthorized
5. `transfer_policy_id` 可以热升级

优点：
1. 协议级共享注册表，一次更新所有 token 生效。
2. 接口标准化。
3. 支持不对称约束。
4. Precompile gasfee/性能有优势。

##### RBAC(TIP-20 内置)

TIP-20 把"管理 token"这件事拆成 5 个职责清晰的 role：DEFAULT_ADMIN（根管理）/ ISSUER（印钱）/ PAUSE（紧急停）/ UNPAUSE（恢复）/ BURN_BLOCKED（合规没收）。

```
┌──────────────────────────────────────────────────────┐
│  USDC TIP-20 token 部署后的 role 分配                   │
├──────────────────────────────────────────────────────┤
│                                                       │
│  DEFAULT_ADMIN_ROLE                                   │
│    ├─ 持有者: 4-of-7 多签（高管或法人实体）              │
│    ├─ 冷钱包，离线签名                                 │
│    └─ 几乎不调用，每年仅几次                           │
│                                                       │
│  ISSUER_ROLE                                          │
│    ├─ 持有者: 国库系统或自动化发行基础设施                │
│    ├─ 热钱包（HSM 保护）                               │
│    ├─ 需要 mint 时实时调用                             │
│    └─ 每天数千笔操作                                   │
│                                                       │
│  PAUSE_ROLE                                           │
│    ├─ 持有者:                                         │
│    │   - SOC 自动化监控系统                            │
│    │   - 安全工程团队 2-of-3 多签                      │
│    └─ 24/7 待命，分钟级响应                            │
│                                                       │
│  UNPAUSE_ROLE                                         │
│    ├─ 持有者: CEO + CISO + 法务 3-of-3                 │
│    ├─ 跟 PAUSE 完全不同的人和钱包                       │
│    └─ 仅在事故确认安全后人工调用                       │
│                                                       │
│  BURN_BLOCKED_ROLE                                    │
│    ├─ 持有者: 合规团队多签 + 法务确认                   │
│    ├─ 调用前必须有书面合规依据                          │
│    │   （法庭命令、OFAC 制裁公告、内部合规裁定）          │
│    └─ 每次调用都有 audit trail                         │
│                                                       │
└──────────────────────────────────────────────────────┘
```

* DEFAULT_ADMIN_ROLE
    1. 根管理员，授予/撤销任何角色，改变某 role 的 admin。
    2. 调用所有"管理类"原生方法：切换到新的合规策略，提高供应上限等。
* ISSUER_ROLE
    1. mint/burn
    2. 销毁自己的代币也需要 ISSUER_ROLE，burn: user -> ISSUER_ROLE -> burn.
* PAUSE_ROLE
    1. 立即停止所有该 token 的转账，监测到攻击/法律命令冻结 /合约 bug 等。
* UNPAUSE_ROLE
    1. 仅且仅能解除 pause 状态
    2. PAUSE 给"低门槛快速响应"组（自动监控、SOC），UNPAUSE 给"高门槛人工审核"组（高管多签），任一被攻陷，攻击者只能拿到单向能力，不能完成"先停再放"的 cycle
* BURN_BLOCKED_ROLE
    1. 强制销毁任意地址的 token，前提是该地址被 TIP-403 policy 拒绝
* UNGRANTABLE_ROLE
    1. 不是业务角色，是协议自身用的占位/防御角色。
    2. 设置某 role 的 admin 为 UNGRANTABLE，该 role 永远再也不能被授予。

##### TIP-1006(In Review + TBD)

把"销毁任意地址余额"这个能力从"必须先拉黑"的双保险，改成"只看角色"的单保险——主要是为了让跨链桥能直接 burn。

```
Step 1: Alice.approve(bridge_contract, 100 USDC)
        ↓
Step 2: bridge_contract.transferFrom(Alice, bridge_contract, 100)
        ↓ (现在 bridge_contract 持有 100 USDC)
Step 3: bridge_contract 调 USDC.burn(100) 销毁
        但是！bridge_contract 必须有 ISSUER_ROLE
        这是个超级敏感的角色，给 bridge 太危险
```

```
Step 1: bridge_contract.burnAt(Alice, 100)
        ↓
        前提：bridge_contract 持有 BURN_AT_ROLE
        Alice 不需要 approve；不需要预先 transfer
        token 直接从 Alice 余额消失，全网总供应减 100
```
新增 BURN_AT_ROLE 角色，目前讨论的焦点：安全（bridge 被攻击），合规。

##### 合规系统

整个合规栈被设计为一个**整合系统**，而不是四个独立的特性。一个受监管的欧元稳定币发行方在 Tempo 上的工作流通常是这样：

1. 部署一个 TIP-20 代币，声明 `currency = EUR`。
2. 附加上他们的 TIP-403 白名单策略（已通过 KYC 的地址列表）。
3. 把 `ISSUER_ROLE` 分配给国库系统、`PAUSE_ROLE` 给运营团队、`BURN_BLOCKED_ROLE` 给合规团队。
4. 每一笔转账自动携带用于 ERP 对账的 memo。
5. 一旦出现被制裁的对手方，合规团队通过 `BURN_BLOCKED_ROLE` 销毁其余额。
6. 当他们再发行第二只稳定币（比如代币化的欧元存款）时，**直接继承同一份 TIP-403 策略**——不需要重复配置，一次策略更新覆盖两个资产。

整套合规栈是**协议原生、在预编译层一次性审计、并跨所有发行方标准化的**。要在 BSC 上做出等价物，则需要每家发行方部署自定义合约、对每项功能给出定制实现，且没有任何跨发行方的标准化基础可供合规工具依赖。


#### Yield 层

TIP-20 的 yield 系统针对计息稳定币，首次做成 token 标准的协议级原语。全局只维护一个累加器（global_reward_per_token），每个用户只在状态变化时同步快照——任何时刻都能 O(1) 算出应得利息。ERC-20 时代每家发行方各自发明的"链下 + 空投 / rebase / wrap / 每笔 transfer 计息"四种 hack，TIP-20 一次性用一个标准化算法替代。

1. Step 1：部署 USDM (TIP-20)
    1. 全局 rpt = 0，opted_in_supply = 0
2. Alice mint 100 USDM、Bob mint 200 USDM
    1. balance[Alice] = 100, balance[Bob] = 200
    2. 两人 reward_recipient = 0（默认未 opt-in）
3. Alice opt-in
    1. opted_in_supply: 0 → 100
    2. Alice.reward_per_token = 0 (snapshot)
    3. Alice.reward_recipient = Alice
4. 发行方第一次发利息
    1. Tether.distributeReward(50)
    2. 合约余额 +50 USDM（托管），delta_rpt = 50 / 100 (opted_in) = 0.5，全局 rpt: 0 → 0.5
5. Bob opt-in
    1. Bob.setRewardRecipient(Bob)
    2. 先 update_rewards(Bob): delta=0.5 但 cached_delegate=0 → 不累计旧的
    3. Bob.reward_per_token = 0.5 (snapshot, 跳过之前的)
    4. opted_in_supply: 100 → 300
4. 发行方第二次发利息
    1. Tether.distributeReward(60)
    2. 合约余额 +60 USDM，elta_rpt = 60 / 300 = 0.2，全局 rpt: 0.5 → 0.7
5. 用户 claim
    1. Alice：delta = 0.7 - 0 = 0.7，reward = 100 × 0.7 = 70，合约转 70 USDM → Alice
    2. Bob：delta = 0.7 - 0.5 = 0.2，reward = 200 × 0.2 = 40，合约 40 USDM→ Bob

#### Tempo 集成层

TIP-20 不是孤立的 token 标准，它跟 Tempo 协议的其他子系统深度集成，形成一个"环绕 TIP-20"的能力网络。

##### Fee AMM：任意 USD 付 gas

让用户用任何 USD 系 TIP-20（USDC / USDT / pathUSD ...）付 gas，协议自动通过 AMM 兑换成验证者的偏好币种。

1. 交易里指定 fee_token = USDC（目前必须是 USD 系列）
2. TIP-20 的 transfer_fee_pre_tx(from, amount) 被协议调用，从用户扣 fee
3. 如果跟 validator 偏好币不同，走 Fee AMM 兑换（固定 1:0.997）
4. 交易完成后 transfer_fee_post_tx(to, amount) 退款多余 gas

* 当前主网（仅支持 USD ↔ USD 付 gas），
1. user 付 USDC  ── [Fee AMM] ─→ validator 收 USDT

* 未来 Onchain FX (开发中，跨币种)
1. user 付 EURC  ──→ Stablecoin DEX ──→  Fee AMM ──→ validator 收 USDT
2. USD ↔ USD 汇率稳定 1:1，跨币种需要专业的做市商处理"市场价格"。

* Fee AMM Pool
1. 任何人都可以提供 USDC ↔ USDT 的流动性。
2. 用户每次付 GasFee 时，如果需要 Fee AMM 进行兑换，会收取 30bps 手续费。
3. 当池子失衡时，套利者的反向 swap，优惠 15 bps。
4. 用户付 30 bps fee（小，可接受），LP 净赚 15 bps（被动收益），套利者抽 15 bps（市场维护服务费）。

实际上 validator-LP 有信息优势主导收益。LP 只存 validator_token，被动收 30 bps fee 中的 15 bps；另 15 bps 留给套利者——他们调 rebalance_swap（反向兑换，0.9985 比率）维护 pool 平衡。双层 fee（30 bps 入 pool + 15 bps 出 pool）形成完美的"用户付费 → LP 收益 → 套利者维护"三方激励循环。

##### Stablecoin DEX：稳定币订单簿

Tempo 自带的订单簿（CLOB）DEX，让所有 USD 系 TIP-20 之间能高效互换。TIP-20 通过 quoteToken 字段告诉 DEX "我跟谁配对"。

1. TIP-20 部署时设 quoteToken = pathUSD
2. 所有引用 pathUSD 为 quote 的 token 自动构成星形流动性图
3. DEX 在内部为每对 (base, quote) 维护订单簿
4. 任意 token 互换走两跳：A → pathUSD → B

链上完整的限价订单簿——maker/taker 模式跟传统证券交易所一模一样，搬到了链上。

##### Virtual Addresses（TIP-1022）

让交易所/支付商给每个客户分配独立入金地址，而无需为每个地址付 250K gas 创建账户、也无需 sweep 交易。

1. Binance 一次性注册:
  调 AddressRegistry.registerVirtualMaster(salt)
  → 得到 masterId = 0x07A3B1C2
  → 这个 masterId 跟 0xBinance 永久绑定（一笔上链）
2. 之后给用户分配地址(链下):
  Alice → 0x07A3B1C2 FDFDFDFDFDFDFDFDFDFD A1A1A1A1A1A1
                     magic                Alice 的 ID（Binance 自定义）
3. Alice 转 100 USDC → 0x07A3B1C2FDFDFDFD...A1A1A1A1A1A1:
    1. Tempo 协议检测中间 10 字节是 0xFDFD..FD magic
    2. 提取前 4 字节 = masterId = 0x07A3B1C2
    3. 查注册表：masterId 0x07A3B1C2 → 0xBinance
    4. 100 USDC 直接打入 0xBinance 余额
    5. emit 两个 Transfer 事件
        1. Transfer(Alice, 0x07A3B1C2..A1A1A1A1A1A1, 100)  ← 用户角度
        2. Transfer(0x07A3B1C2..A1A1A1A1A1A1, 0xBinance, 100) ← 转发角度
    6. Binance 后台监听事件
        1. 看到 Transfer 中间地址含 magic
        2. 解析 userTag = A1A1A1A1A1A1 → "这是 Alice 的"
        3. Binance 内部账本: Alice +100 USDC

* 限制
1. 仅 TIP-20 转账有效；普通 ERC-20 转到虚拟地址会丢失
2. 仅以 transfer 路径有效；setRewardRecipient 拒绝 virtual address

##### Access Keys: 限额 / 调用域(TIP-1011)

* Access Keys 让 Tempo 解锁了消费级 UX 和 AI agent 经济两个关键场景
    1. 消费级：用户用 Face ID / Touch ID 签名，不再管助记词
    2. AI agent：用户给 agent 限额 + 调用域，最坏情况损失被锁在限额内
* Access Keys 把 ERC-4337 的用智能合约做钱包范式提升到了'协议层 native'
    1. 三个权限维度（spending limits / call scope / calldata recipient allowlist）
    2. 三种签名类型（secp256k1 / P256 / WebAuthn）
    3. 配套 TIP-1020（链上签名验证 precompile）

```
┌─────────────────────────────────────────────┐
│                                              │
│   Root Key（你的主私钥）                       │
│   ├─ 等价于传统私钥                           │
│   ├─ 全权限                                   │
│   ├─ 应该藏在冷钱包                            │
│   └─ 用来：                                   │
│       1) 大额操作                              │
│       2) 授予 / 撤销 access keys                │
│                                              │
│   ↓ 通过 authorizeKey() 派生                  │
│                                              │
│   Access Keys（子钥匙）                        │
│   ├─ 受限权限（你定义）                         │
│   ├─ 可以是 P256 / WebAuthn 等                │
│   ├─ 应该用来：                                │
│   │   - 钱包热签名                             │
│   │   - AI agent 子钥匙                        │
│   │   - 订阅服务授权                           │
│   │   - 子账户                                 │
│   └─ Root 可随时撤销                           │
│                                              │
└─────────────────────────────────────────────┘
```

**Access Keys 通过 Tempo Native AA 实现**

* 三大权限维度
    1. Token Spending Limits: 哪些 TIP-20 token, 限额, 周期
    2. Call Scope：哪个合约，哪些方法，收件人白名单（仅 transfer 类用）。
    3. Calldata：协议层会解析 calldata 的 ABI 第一个 address 参数，跟白名单比对（transfer， approve， transferWithMemo）。
* 签名类型支持
    1. secp256k1：需要助记词 / 私钥
    2. P-256 + WebAuthn：消费级 UX 
    3. TIP-1020：P-256 / WebAuthn 验证不是 EVM 原生的，所以 Tempo 引入了 SignatureVerifier precompile。

* 场景
    1. AI Agent（如 Cursor）替你买 API
    2. 订阅服务（每月扣款）
    3. 消费级钱包（用 Face ID 而非助记词）
    4. DEX 限额仓位

##### Protocol AA: Fee sponsorship/Batch/Passkey/Scheduled

Tempo 有个新的交易类型 0x76（EIP-2718 兼容）。仅支持如下 6 中

| 能力 | TIP-20 场景 |
| --- | --- |
| Fee Sponsorship | 商户替用户付 gas → 用户钱包完全不需要任何 token 就能调 USDC.transfer |
| Batch Transfer | 一笔交易调多个 TIP-20 操作（如发工资 1000 笔 transfer），原子性，全成功或全失败 |
| Scheduled Transfer | 协议级时间窗口（valid_after / valid_before），无需链下 cron job |
| Passkey 签名 | TIP-20 transfer 用 Face ID / Touch ID 签名，无需助记词 |
| 2D Nonce | 每账户多条并行 nonce 序列 → 多个并行 transfer 不互相阻塞 |
| EIP-7702 Authorization | 临时升级 EOA 为合约，组合任意 TIP-20 操作 |

* Fee Sponsorship
    1. ERC-4337 依赖 paymaster。
    2. `TempoTransaction (0x76)` 增加 fee_payer 和 fee_payer_signature 两个字段。
    3. 流程：
        1. 用户把签好的 intent 发给 fee_payer 服务
        2. fee_payer 服务收到，决定是否赞助
        3. fee_payer 的 nonce: 不增加！
        4. fee_payer 的签名 digest 包含完整 tx
    4. 场景：Shopify 商户结账，公司给员工发工资，新用户 onboarding

* Batch Transfer
    1. 协议级零额外开销——calls 直接由 EVM 执行器遍历，不经过任何中间合约
    2. TempoTransaction.calls 直接是个 Vec<Call>。

* Scheduled Transfer
    1. TempoTransaction 增加valid_after 和 valid_before 字段
    2. mempool 层进行控制，时间未到达跳过，过期则丢弃。

* 2D Nonce
    1. 账户的 nonce 是 Mapping<nonce_key, u64>
    2. 钱包同时有 3 个并行操作: 用户在 A 应用付款 / 自动续费订阅 / AI agent 调 API
    3. Subblock 路由的 nonce_key，第一字节是 magic prefix + 后 15 字节是 validator partial pubkey。
    4. Expiring Nonces：tx hash 作为 nonce + valid_before 过期

##### Permit (EIP-2612): 免 gas 授权(TIP-1004)

TIP-1004 给所有 TIP-20 token 强制内置 EIP-2612 permit 功能——用户链下签名就能授权 spender 花费 token，无需先发链上 approve 交易。核心价值：
1. 让 DApp 能把 approve+action 打包成 1 笔交易（UX 跟 Web2 拉平）；
2. 让"sweep 从未发过 tx 的地址"成为可能（Fee Sponsorship 解决不了这个）；
3. 协议级统一实现（Tempo 所有 TIP-20 都支持），而以太坊上每个 ERC-20 各自实现各有差异。

这是 TIP-20 集成层最简单但实际生态影响最大的特性，跟以太坊主流标准（EIP-2612）100% 对齐的 TIP。

##### Fee Token Introspection: 合约查 fee token(TIP-1007, In Review)

让智能合约能查询"本笔交易实际用的什么 token 付 gas"——支持折扣、退款、多币种结算等场景。

Paradigm CTO 提案，Tempo 的某个合作伙伴（很可能是 Stripe / Visa / Bridge）主动要求这个特性。
可能场景：我们要给用户一些福利，如果用户用 PathUSD 付 gas（默认便宜），我们额外回 0.5% 折扣给用户。

##### TIP20Factory: token 部署

专门的 precompile（地址 0x20FC...）负责部署新 TIP-20 token——发行方不直接部署字节码，而是调用 factory。

### TIPs 

```
┌─────────────────────────────────────────────────────────────┐
│ 层 6: Tempo 集成层                                           │
│  ├─ TIP-1004  EIP-2612 Permit (gasless approval)            │
│  ├─ TIP-1007  Fee Token Introspection (合约查当前 fee token) │
│  ├─ TIP-1022  Virtual Addresses (自动转发到 master wallet)    │
│  ├─ TIP-1011  Enhanced Access Keys (用 TIP-20 selector 限权) │
│  └─ TIP-1010  Gas Parameters (定义 TIP-20 transfer 50K gas)  │
├─────────────────────────────────────────────────────────────┤
│ 层 5: Yield 层                                               │
│  └─ (TIP-20 自带，无独立 TIP)                                  │
├─────────────────────────────────────────────────────────────┤
│ 层 4: 合规层                                                 │
│  ├─ TIP-403   Policy Registry (基础合规策略系统)              │
│  ├─ TIP-1015  Compound Transfer Policies (sender/recip 三策略) │
│  ├─ TIP-1006  Burn At for TIP-20 (合规销毁) in review        │
│  └─ TIP-1047  Reject code creation in TIP-20 prefix (保护地址空间) │
├─────────────────────────────────────────────────────────────┤
│ 层 3: 货币声明层                                              │
│  └─ (TIP-20 自带，ISO 4217 immutable)                        │
├─────────────────────────────────────────────────────────────┤
│ 层 2: 对账层 (Memo)                                           │
│  └─ (TIP-20 自带，transferWithMemo / mintWithMemo / etc.)    │
├─────────────────────────────────────────────────────────────┤
│ 层 1: ERC-20 baseline                                        │
│  └─ TIP-20 自带（兼容 ERC-20 接口）                             │
└─────────────────────────────────────────────────────────────┘

支撑性 TIP（不在某一层，但影响 TIP-20 实际成本/性能）:
  ├─ TIP-1000   State Creation Cost Increase (TIP-20 SSTORE 价格)
  ├─ TIP-1010   Mainnet Gas Parameters (定义 base fee + 容量)
  └─ TIP-1016   Storage Reservoir (技术细节)

DEX 相关（用 TIP-20 但不属于 TIP-20 自身）:
  ├─ TIP-1001~1005, 1030  Stablecoin DEX 订单簿设计
  └─ TIP-20 的 quoteToken / nextQuoteToken 是 DEX 集成接口
```

## MPP（Machine Payments Protocol）

MPP (Machine Payments Protocol) = HTTP 协议扩展，把 HTTP 规范里那个一直没用上的 402 Payment Required 状态码激活并标准化，让机器/AI agent 直接在 HTTP 请求层面完成付费 + 调用——支付被"内联"进 HTTP 请求，无需账户、API key、KYC。

###  MPP 三层架构

```
┌─────────────────────────────────────────────────────┐
│                                                        │
│  应用层 (HTTP)                                          │
│  ─────────────────────────────                         │
│  • HTTP 402 Payment Required 状态码                     │
│  • X-Payment Headers (signed payment credentials)      │
│  • 请求 / 响应内联付费                                   │
│  ↓                                                     │
├─────────────────────────────────────────────────────┤
│                                                        │
│  会话层 (Session)                                      │
│  ─────────────────────────────                         │
│  • Channel-based payment streaming                    │
│  • 一次 open + 多次 off-chain payment + 一次 close      │
│  ↓                                                     │
├─────────────────────────────────────────────────────┤
│                                                        │
│  链上锚定层 (Blockchain)                                │
│  ─────────────────────────────                         │
│  • TempoStreamChannel Solidity 合约                    │
│  • open(payee, token, deposit, salt, authSigner)      │
│  • close(channelId, cumulativeAmount, signature)      │
│                                                        │
└─────────────────────────────────────────────────────┘
```

#### ITempoStreamChannel
    1. payee 接收方（API 服务方）
    2. token 用什么 token 付款（必须 TIP-20）
    3. deposit 一次性存入的金额（封顶）
    4. authorizedSigner 谁能签 off-chain payment（通常是 agent 的 ephemeral key）
    5. 返回 channelId 唯一通道 ID。

#### WorkFlow
    1. Agent 想第一次调 Anthropic API。
    2. Anthropic 服务端响应 402
    3. 链上 open channel （第一次），返回 channelId
    4. Anthropic 看到链上事件，准备好接收 off-chain payment
    5. 链下
        1. Agent 重发原请求带 payment header（channelId，预算等）
        2. Anthropic 验证签名，返回 200
        3. Agent 多次调用 Anthropic 服务。
    6. 链上：
        1. Anthropic 决定结算（每天 / 每周 / channel 快用完时）
        2. Anthropic 提交 tx: TempoStreamChannel.close。
        3. 结算并退还剩余 token。

### MPP + Access Keys
```
1. 用户 root 给 AI agent 一把 access key:
   - 限额: $200/月
   - 允许调: TempoStreamChannel.open / close
   - target 白名单: [Anthropic, OpenAI, Pinecone]

2. AI agent 用 access key open 一个 MPP channel:
   open(Anthropic, deposit=$50) ← 受 access key 约束 ✓

3. Channel 内 1000 次 off-chain 调用:
   不再受 access key 约束（已经在 channel 里了）

4. 月底 close channel
```

## Onchain FX（开发中）

如今的跨境支付使用 Tempo 所谓的**"稳定币三明治（stablecoin sandwich）"**：链上那段是 USD 稳定币，两端各靠一次链下 on/off-ramp 把本地货币转换进/出。这一过程依赖传统 FX 提供方，引入结算延迟与费用，并把可触达市场限制在以美元为中心的走廊内。

Tempo 正在构建链上原生外汇机制——通过一组受监管的非美元稳定币发行方与 DEX 流动性池，实现稳定币之间的直接兑换（USD↔EUR、USD↔SGD 等）。一旦上线，链下兑换段将被消除，从而实现完全链上、多币种的支付流动。多币种手续费支付也将随之就位——欧洲用户用 EURC 付 gas，新加坡用户用 SGDR 付 gas。

TIP-20 的货币标识符字段（ISO 4217，部署时不可变） 是这一机制的基石——协议天然就知道每个代币代表的真实世界货币是哪一种，从而可以无需预言机或人工配置就完成自动化路由。

现状：基础设施已部署，但未接入费用流程。CLOB（中心限价订单簿），tick-based，参考 Hyperliquid / Aptos Econia，任意 TIP-20 稳定币之间（USD/EUR/SGD 跨币种 OK）。

## Privacy / Zones（开发中）

暂时披露的资料不多,明确信息：
1. Optional / Opt-in：不是默认隐私，而是用户选择
2. 协议级：不是应用层加密，是协议提供原语
3. 合规友好：必须保留对监管的"selective disclosure"（选择性披露）

### Tempo Zones 产品定位

Zone: 一条 EVM 兼容的私有链，与 Tempo 主网并行运行。

#### Zone 内 
1. Zone 运营者（operator）：能看到 zone 内所有交易（合规）
2. 用户本人：只能看到自己的交易和余额
3. 其他人：只看到 zone 状态有效性的密码学证明，看不到任何细节。
4. 资金锁在主网的 zone 合约里，只能被资产所有者本人提走。
5. 合规规则会跟着 token 走
    1. 发行方在主网更新黑名单/冻结 token, 所有 zone 自动同步生效

参与者在 zone 内交易对外完全不可见。资产可以跨 zone 互通——目前的设计都必须经过主网。

#### 入金
1. 主网上只暴露：token、金额、发送方
2. 接收方地址被加密——只有发送方和 zone operator 能解密
3. memo 也被加密

#### 出金
1. 出金者地址通过**密码学承诺（cryptographic commitments）**隐藏
2. 只有接收方能识别真实身份
3. 主网上看到"有钱从某 zone 出来到某地址"，但看不到 zone 内是谁发的

### Tempo Zones 目前技术侧进展

1. 访问控制：Zone 内 TIP-20 非用户本人或者sequencer不能查询，RPC 需要用户签名才能访问。
2. 出入金：会有加密（承诺方案 + 公钥加密）。
3. 证明系统：ZKVMs and TEEs， 用户 Zone -> 主网 状态提交。
4. Zone 内数据是否加密，用什么方式加密并没有说（如果是明文，对于企业维护 zone sequencer 也能满足）。

### 安全假设

| Sequencer 安全假设 | 失信会怎样 |
| --- | --- |
| Sequencer for liveness | sequencer 停了 → 整个 Zone 停摆 |
| Sequencer for inclusion and ordering | 你的交易（包括提现）会被排除或重排 |
| Sequencer for privacy | sequencer 能看到 Zone 内所有交易 |
| Sequencer for data | 没有 sequencer，Zone 状态无法重建 |
| Sequencer + verifier for correctness | 如果 verifier / 证明系统有严重 bug + sequencer 恶意 → 可能被偷钱 |

### Reference
1. https://tempo.xyz/blog/privacy-on-tempo
2. https://tempo.xyz/blog/introducing-tempo-zones
3. https://docs.tempo.xyz/protocol/zones
4. https://docs.tempo.xyz/protocol/zones/proving

## HardForks

| Hardfork | 块号 | 激活时间（UTC） | 主要内容 |
| --- | --- | --- | --- |
| Genesis | 0 | (epoch 0) | 主网创世 |
| T0 | 0 | (epoch 0) | 默认 fork；base fee = 10 Gwei attodollars/gas |
| T1 | 4,494,230 | 2026-02-12 15:00:00 | TIP-1009 过期 nonce + TIP-1010 主网 gas 参数（base fee 升 20 Gwei、general 限额固定 30M、block budget 500M） |
| T1A | 4,494,230 | 2026-02-12 15:00:00（与 T1 同 block） | 移除 EIP-7825 单笔 gas 上限（16.7M → 30M），配合 TIP-1000 让大合约部署可行 |
| T1B | 6,253,936 | 2026-02-23 15:00:00 | 修复 / 调优 bundle（源码无单独描述，含微调） |
| T1C | 8,967,991 | 2026-03-12 15:00:00 | 修复 / 调优 bundle |
| T2 | 12,286,033 | 2026-03-31 14:00:00 | TIP-1004 Permit、TIP-1015 复合转账策略、TIP-1017 Validator Config V2、TIP-1036 T2 Bug Fixes、2D nonce gas 调整 |
| T3 | — | 2026-04-27 14:00:00 | TIP-1011 增强访问密钥、TIP-1020 签名验证 precompile、TIP-1022 Virtual Addresses、TIP-1038 T3 Bug Fixes |
| T4 | — | ❓ 未排期 | TIP-1031 Embed consensus context（Approved）—— 共识上下文嵌入区块头 |
| T5 | — | ❓ 未排期 | 暂无 TIP 明确归属 |

## BSC/NewChain
* 技术侧
    1. payment-lane -> Mulit-lanes
    2. Rust 原生 transfer -> EVM 字节码？
    3. 继承 Tempo Native AA 模板？
* 产品侧
    1. 对账 memo.
    2. 打造一套合规体系, 支持 RBAC 非稳定币的 BEP-20+?
    3. Access Keys + MPP 支持 AI Agent(需要支持 BNB -> USD 结算)?


























