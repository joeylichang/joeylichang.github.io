# Data Availability

## What is Data Availability on Layer2
数据可用性在 Layer2 的方案中备受关注，因为 Layer2 作为 Layer1 的扩展层，无论扩展的是 Layer1 的执行能力还是存储能力，都不会在 Layer1 通过执行 Layer2 全部的区块交易来保证数据可用性，因为那样的话失去了 Layer2 的扩展意义。当然 Layer1 也是存在数据可用性的，但是由于 Blockchain 去中心化中的无权限性导致其天然支持，自然在在 Layer1 上就没有 Layer2 那么值得讨论。

在传统的分布式存储中的数据可用性主要强调的是可访问，默认是数据正确（一般存储不会也没有动力恶意篡改数据，不正确多数是因为 bug 导致，所以讨论的不多）。在 Blockchain 中由于去中心化性（传统的分布式存储都是基于运营商中心化的系统），数据可用性除了关注可访问性，更关注正确性。防止先入为主的观念导致概念或者讨论的内容不清晰，这里有必要明确一下。

下面看一下 Blockchain 中数据可用性中的可访问性和正确性具体指的是什么？

Blockchain 的去中心化性中的无权限性明确的指出任何节点或者用户不需要成为系统的一部分就可以读取和写入数据，其中一层含义就是任何账户都可以访问数据，这似乎是天然支持的，但是在 Layer2 中由于 Layer1 不会完全执行区块交易，Layer2 可能会隐瞒部分数据，所以 Layer2 的数据可用性中的可访问性主要关注的是如何保证 Layer2 广播全量数据而没有隐藏。

数据的可访问性得到保证之后，其实数据就具备了可被验证性，其他节点通过验证交易即可保证数据的正确性。但是针对 Layer2 这似乎也不够用，因为如果发生了恶意篡改数据的情况，但是由于数据已经上链了，需要识别并在 Layer1 上进行回滚， 甚至需要对这种情况进行负向激励（惩罚作恶），所以需要有机制保证对于篡改数据提出挑战（欺诈证明）并行处理。同时还需要保证数据不能被少数和别节点控制甚至篡改，因为 Layer2 只会复用 Layer1 的去中心化和安全性数据的编排还是由 Layer2 负责，需要保证不能被恶意控制。

* 数据可用性中的可访问性，需要保证 Layer2 广播全量数据而没有隐藏。
* 数据可用性中的正确性包括：数据可验证，防欺诈。

下面主要针对上面两点的内容进行展开的说明。

## Start From Fraud Proofs
为了更深刻的理解数据可用性，可以从欺诈证明作为切入点更深刻的体会一下（上面只是简单的概要性介绍，本部分深入到实际的场景进行讲解）。

假设攻击者创建了一个在某种方式上无效的区块（状态根无效或者交易无效等）。可以创建一个包含该交易以及默克尔证明的“欺诈证明”，并用它来说服任何轻客户端以及区块链本身，证明该区块是无效的。验证欺诈证明涉及使用给定信息尝试实际处理区块，并查看处理过程中是否存在错误；如果有，则该区块无效。

欺诈证明理论上允许轻客户端获得几乎与完整客户端等价的保证，以确保客户端收到的区块的正确性（“几乎”是因为机制的额外要求是轻客户端必须与至少一些其他诚实节点连接）：如果轻客户端收到一个区块，则可以询问网络是否有任何欺诈证明，如果没有，则在一段时间后，轻客户端可以假设该区块是有效的。

尽管这一机制优雅，但它存在一个重要漏洞：如果攻击者创建了一个（可能有效，也可能无效）但未 100% 发布的数据区块会怎样？即使可以使用简洁的零知识证明来验证正确性，攻击者如果成功发布不可用的区块（缺失）并使其被包含在链中，这仍然是不好的，因为这种情况会剥夺所有其他验证者完全计算状态的能力，或创建与不再可访问的状态部分交互的区块。接受包含未发布数据的有效区块也有其他不良后果（原子跨链交换或数据交易的哈希锁等方案都依赖于发布到区块链的数据将是公开可访问的这一假设）。

欺诈证明并不能完全解决上述问题，因为攻击者完全可以延迟发布来反驳欺诈。假设攻击者进行一种攻击，隐瞒数据，等待“挑战者”抓住他们，然后立即发布数据。在这种情况下：
1. 挑战者的奖励是净正的吗？如果是这样，恶意的挑战者发出虚假警报就是一个资金泵漏洞。
2. 奖励是零吗？如果是，这就是一种无成本的拒绝服务（DoS）漏洞，可能被利用来迫使节点下载区块链中的所有数据，从而使轻客户端或分片的好处无效。
3. 奖励是负的吗？如果是，那么作为渔民就是一种利他行为，因此攻击者只需在经济上消耗掉利他主义的渔民，就可以自由地在没有阻碍的情况下包含数据不可用的区块。

上述内容说明了**欺诈证明并不是保证数据可用性的全部方案，欺诈证明只能是保证数据正确性，但是数据的可访问性是无法保证的**，需要额外的内容一起组成数据可用性方案，后续会介绍相关方案，例如 EC/RS 编码基于概率的方案等。

## Validity Proofs vs. Fraud Proofs
前一部分为了深入强调数据有效性的包含含义（数据正确性和可访问性），从欺诈证明的角度展开并阐述了欺诈证明只是数据可用性方案的一部份，不是等价关系。上述的切入点需要对 Layer2 有一些了解（熟悉或者理解欺诈证明的常规方案，但是证明数据正确性的方式不仅仅是欺诈证明。还有一种方式是有效性证明，由于其性能问题目前应用并没有 Optimistic Rollup 的欺诈证明那么被广泛应用（V 神认为基于 ZK 的有效性证明是未来），但是考虑到知识的完备性，这里会进行对比性的介绍。

### Fraud Proofs
欺诈证明提供了状态转换不正确的证据。它们反映了一种乐观的世界观：假设区块仅代表 L2 数据的正确状态，直到有证据证明相反。实际上，提交的区块可能确实包含一个不正确的状态转换。欺诈证明的主要优点在于，它们不需要应用于每一个状态转换，而仅在发生错误时才需要。因此，它们需要的计算资源较少，更适合在可扩展性受限的环境中使用。这些协议的主要缺点源于其互动性：它们定义了多个参与方之间的“对话”。对话需要各方——特别是声称欺诈的一方——在场（活跃性），并允许其他方通过各种方式打断对话。但问题的核心在于协议将沉默（对新状态缺乏挑战）的解释视为隐含同意。实际上，攻击者可能会试图通过 DDoS 攻击制造沉默的假象。

### Validity Proofs
有效性证明提供了状态转换正确的证据。它们反映了一种更悲观的世界观。区块仅在状态正确的情况下包含表示 L2 状态的值。有效性证明则要简单得多：将某些链外计算的结果发送到智能合约。智能合约仅在验证该值为正确后，更新区块链。有效性证明的主要优点是，区块链将始终反映正确的 L2 状态，新的状态可以立即被依赖和使用。主要缺点是每次状态转换都需要提供证明，而不仅仅是在状态转换受到质疑时，这影响了可扩展性。

### 51%-Attacks
51% 攻击在两种证明方式中是很重要的区分点，对于欺诈证明使用 Dispute Time Delay (DTD) 来加大 51% 攻击的成本从而保证安全性，但是理论上这种攻击还是行得通的（可能随着交易的频繁，这个成本相对收益会很诱人），所以在欺诈证明的防攻击讨论中 DTD 默认7天，之后不再会讨论这种攻击，或者说防 51% 攻击不再是欺诈证明的讨论范围了。

有效性证明，确是可以防止 51% 攻击，即使 L1 被攻击，写入 L1 的 L2 数据依然是正确的，即使 L2 的数据可能发生会滚（由于 L1 倍攻击），但是数据的正确性是可以保证的，不存在写入无效交易和任何盗取，显然安全性更高。

## Data Availability Design
有了前面的说明，应该可以清晰数据有效性包含：数据可访问，数据可验证，无欺诈，这三个基本的内容，但是目前还没有一个完整的方案可以保证这些，本部分重点介绍完整的数据可用性系统。

### Basic Knowledge
为了更好的理解数据可用性系统的设计，需要对模型，场景甚至是目标进行更细粒度的描述，好处是对于需要的介绍内容划清了边界以及所需要的背景知识（如果对于 Optimistic Rollup 比较了解，会更容易理解这部分基础知识的介绍）。

#### Light Client
考虑到链上容量轻客户端是一个很好的选择，因其需要更少的数据更低廉的硬件要求。这就提出了一个要求，即轻客户端如何保证数据的可用性，或者说轻客户端如何确认数据的可用性。

我们先来看一下全节点的数据可用性，全节点需要全两数据并通过执行区块交易来验证数据的可用性（需要出块节点广播全量数据，并进行了验证，所以保证了数据可访问性和可验证行）。

对于轻客户端来说，没有全量数据，所以需要一套方案或者证明系统来辅佐轻客户端保证或者确认数据可用性，**数据可用性系统，主要是针对轻客户端进行的设计**。

用于辅佐轻客户端保证数据可用性的方案，在 Optimistic Rollup 中至少需要两部分：
* 有 Layer2 的全节点作为挑战者，发起欺诈证明，告诉轻客户端有造假。
* 有 Layer2 的全节点作为数据基础，让请客户端采用一些手段（概率，纠删码等）保证出块节点广播了全量节点。

#### Assumptions && Target
通过上面基础的数据可用性场景的粗略描述，可以看到数据可用性系统并不是万能的，需要一些假设（比如有可靠的全节点）来描述系统的边界，在这里需要清晰一下。

* 假设至少有一个诚实的全节点愿意生成可以在最大网络延迟内传播的欺诈证明。 
* 假设每个轻客户端至少与一个诚实的全节点连接。
* 假设有最少数量的诚实轻客户端能够从区块中重建缺失的数据（在类似概率性系统设计中，需要足够数量的轻节点来保证数据被全量广播）。
* 无法依赖参与共识的节点中存在一个诚实的多数。
  * 在 Optimistic Rollup 中，数据会上 L1，去中心化和安全性由 L1 保证，所以 L2 的共识被攻击如果有数据可用性系统就可以保证数据是没问题的。
  * 这里强调的不依赖共识，主要指的是 L2 轻客户端不依赖，也就是数据可用性系统，能够在不依赖共识的情况下发现其作恶（数据可用性中的防欺诈）。
  * **即使没有多数诚实共识节点的假设，但是不会发生双花攻击（需要改变最长链），这里的不诚实紧紧是交易的有效性不能被保证**。
* 轻客户端只下载区块头，并假设交易列表是根据交易有效性规则有效的（紧紧处理状态数据，不对交易做处理，但是可以处理欺诈证明的 mpt proof）。
  * 轻客户端根据共识规则验证区块，但不验证交易有效性规则，因此假设共识是诚实的，即它们仅包含有效交易。
  * 轻客户端可以从完整节点接收默克尔证明，以证明特定交易或状态对象已包含在某个区块头中。

数据有效性系统的目标：
* 数据可访问性和正确性：向客户端证明，对于给定的区块头，在小于 O(n) —— n 是交易 T 的个数 —— 的时间和空间复杂度内，证明数据有问题，即欺诈证明被证实。
* 数据防欺诈：确保轻客户端在存在不诚实多数参与共识的节点时，不接受包含无效交易的区块。

总结一下：
* 数据可用性中的正确性：至少一个安全的全节点 + 提出欺诈证明给轻客户端（L1 上的合约本质上是 L2 的轻客户端）。
* 数据可用性中的可访问性：轻客户端的方案，来确认生产者是否广播了全量数据。
* 数据可用性中的防欺诈：至少一个安全的全节点 + 提出欺诈证明给轻客户端。

### Data Availability System
数据可用性里面的正确性（可验证）和防欺诈通过**至少一个安全的全节点，并愿意提出欺诈证明给轻客户端**的假设得到了保证，基于轻客户端为中心的数据可用性系统的设计重点应该在于如何保障数据的可访问性上，即数据没有被生产者隐藏而全量广播出去。

具体描述一下隐藏攻击：一个恶意的区块生成者可以通过不提供计算特定数据所需的必要信息，而只向网络发布区块的头部信息，从而阻止完整节点（即能够验证全部区块内容的节点）生成欺诈证明。区块生成者可能会选择在区块已经被发布很长时间后，才释放实际的数据。这些数据可能包括无效的交易或不符合协议的状态转换，这会导致该区块变成无效区块。如果这种情况发生，这会引起未来区块中的一定交易被撤销或回滚。因此，轻客户端需要确定与数据根（DataRootI）相匹配的数据确实是可用的，也就是说，它们必须有信心知道这些数据可以被网络访问，以确保交易的有效性。轻客户端通常不存储完整的数据，因此其依赖于完整节点提供的数据是非常重要的。

[A note on data availability and erasure coding](https://github.com/ethereum/research/wiki/A-note-on-data-availability-and-erasure-coding)，
在社区里最早提出了基于 EC 编码（分布式存储中比较常见的技术，不在此赘述）的方案，主要的想法：
1. 轻客户端在接受区块为有效之前，先验证该区块的数据可用性，但以概率方式进行。
   1. 以概率的方式下载多数区块数据，保证数据被广播出来了，这样的轻客户端保持在最少数量以上就可以保证所有的数据块都被广播了。
2. 概率的引入，不可以避免少部份数据还是被隐藏，为此引入了 EC 编码为扩展数据也增加一个 MPT Root，做个维度数据以概率的方式进行校验，极大的减小了隐藏数据的概率。

一个参考的具体实现案例：
1. K 份数据，冗余 K 份数据，并计算出扩展数据的默克尔根（DataRootI），每个叶子对应一个份额（2K 份）。
2. 轻客户端从表示该 DataRootI 的默克尔树中随机抽取份额，仅在收到所有请求的份额后才接受该区块。
   1. 如果恶意区块生成者使超过 50% 的份额不可用，以使完整数据无法恢复，则客户端在第一次抽样时有 50% 的机会随机抽取到一份不可用的份额，抽样两次后为 25%，抽样三次后为 12.5%，依此类推。
   2. 使该方案有效，网络中必须有足够多的轻客户端抽样足够多的份额，以便区块生成者需要释放超过 50% 的份额，才会通过所有轻客户端的抽样挑战，从而可以恢复完整区块。

问题：如果扩展数据造假，是无法恢复的，源数据不发布，依然恢复不了
   1. 本质：判定扩展数据无效的欺诈证明实际上是原始数据本身。数据重新编码，以验证其与给定扩展数据之间的不匹配。需要 O(n) 的数据量。
   2. 使用多维编码，证明大小O(√d n)，其中 d 是编码的维度数量。

[Fraud and Data Availability Proofs: Maximising
Light Client Security and Scaling Blockchains
with Dishonest Majorities](https://arxiv.org/pdf/1809.09044)
提出了多位度的 RS 编码方案，解决上述一维的问题，相应的网络/计算资源和复杂度指数级增加。具体方案如下：
1. k * k，水平和垂直，生成两个扩展矩阵，扩展矩阵在计算，生成扩展的扩展矩阵 2k * 2k。
2. 行列算 hash，然后再 hash，获取 MPT Root。
3. 当轻客户端从网络接收到新区块头时，它们应随机抽取 0 < s < (k + 1)² 个不同的份额，并仅在接收到所有份额之后才接受该区块。此外，轻客户端会将它们已收到的份额传播给网络，以便诚实的完整节点可以恢复完整区块。
4. 最小诚实轻客户端的个数与 block 大小，采样次数，k 相关。
5. 多位度保证了扩展数据造假的难度。

[A data availability blockchain with sub-linear full block validation](https://ethresear.ch/t/a-data-availability-blockchain-with-sub-linear-full-block-validation/5503)
提出了一种极端化的思想，数据可用性完全代替了数据可验证，如果数据是可用的数据就是正确的也是可以被验证的，那么只需要手段保证数据是可用的就行无需验证区块，这种系统就类似 Blockchain 的存储系统（File Coin，Storj 等）。这种系统的好处是，其吞吐量通过添加不参与共识或生成区块的节点而增加。如果你增加进行抽样的节点数量，你可以增加最大安全区块大小。

文章中提出了一种基于主权客户端的方案：
1. 客户端定义的任意应用程序，将区块链作为仅用于发布消息的地方。——链外执行
2. 客户端无需执行来自其他应用程序的消息，除非其他特定应用程序被明确声明为依赖项。
   1. 如果有两个应用程序 A 和 B 使用同一链并且彼此“独立”，如果 B 突然变得更受欢迎，应用程序 A 的用户的工作负载基本保持不变，反之亦然。
3. 主权客户端可以被视为在同一链上存在“虚拟”侧链的系统，每个应用程序相关的交易仅需由该应用程序的用户处理。
   1. 类似于特定侧链的用户只需处理该侧链的交易。
   2. 由于所有应用程序共享同一链，所有交易的数据可用性都由同一共识组平等而统一地保证，这与传统侧链不同。
   3. 如果某些应用程序的用户希望更新应用程序的逻辑或状态，他们可以在不需要对链进行硬分叉的情况下做到这一点。
   4. 如果其他用户不跟随更新，则这两个用户组将对同一应用程序有不同的状态，这类似于一种“链外”的硬分叉。
4. 主权客户端需要维护命名空间进行检索和同步数据。

优点：
1. 类似点对点文件共享网络，区块链的交易吞吐量可以通过增加更多不参与区块生产或共识的节点“安全地”提升。
2. 用户可以为他们自己的“主权”应用程序下载和发布状态转换，而无需下载其他应用程序的状态转换。
3. 改变应用程序逻辑不需要对链进行硬分叉，因为应用程序逻辑是在链外定义和解释的。
4. “验证”一个区块（从完整节点的意义上讲）所需的带宽成本为 O(√blocksize + log(√blocksize))，这是基于使用纠删编码和抽样的数据显示可用性的证明。

这个区块链设计旨在确保数据可用性，同时使完整区块的验证过程所需的资源（如时间或计算能力）低于区块大小的线性水平。这种降低验证复杂度的方法可以在保证系统性能的同时，提升数据可用性。因此，它比传统的区块链模型更加高效。


## Fraud Proof Design
接下来讨论一下欺诈证明的实现，这个对于性能和状态模型的选择也是至关重要的。目前常规的欺诈证明（基于账户模型）是基于 MPT 的中间根为基础的，一个 block 的执行过程中固定的交易数量，固定 gas 等维度达到了配置值的时候会生成相应的状态根root，写入 block header，然后挑战者在发起欺诈证明进行挑战的时候提供前一个中间根以及相应的交易，如果不能生成下一个中间根，则表示欺诈成立。

[Data availability proof-friendly state tree transitions](https://ethresear.ch/t/data-availability-proof-friendly-state-tree-transitions/1453) 
提出了类似上述的中间根概念，并给出一个可行性的实践，使用了 SMT 因为他的操作更稳定没有扩展和收缩，其他没有什么特别。

[Towards on-chain non-interactive data availability proofs](https://ethresear.ch/t/towards-on-chain-non-interactive-data-availability-proofs/4602)
在轻客户端做 L1 的智能合约对数据进行验证（如上述方案），保证数据的可用性，使用了类似洋葱路由的方式，这在智能合约中显然是做不到的。洋葱路由匿名性的原因在于协议的密码学方面，而不是“网络层”方面，所以轻客户端放在智能合约中是做不到的。为此上述论文提出中继节点的概念专门做这个事情，潜在的问题是中继节点有一个不工作了，无法证明数据不可用，也没有完善的奖励和惩罚机制。

[On-Chain Non-Interactive Data Availability Proofs](https://ethresear.ch/t/on-chain-non-interactive-data-availability-proofs/5715?ref=fuel-labs.ghost.io)
指出上述方案在落地的时候都存在不可行性，目前现状的还是**侧链通过在主链上提交侧链区块头来借用主链的安全性，但如果侧链操作员隐瞒了无效区块，则资金仍然可能被盗。单纯地做到这一点需要将所有交易数据发布到链上以保证数据的可用性（以便始终可以计算潜在的欺诈证明）**。为此提出了一个离线的方案，有一个智能合约像中间节点发起请求对数据进行检查，在D个block之后查看结果，如果没响应或者失败则认为数据有问题，但是这个中间系统并没有详细的介绍（只介绍了接口）。

[State Providers, Relayers - Bring Back the Mempool](https://ethresear.ch/t/state-providers-relayers-bring-back-the-mempool/5647)
前面的方案随着对中继系统的奖励和惩罚机制的会引入，也就引入了中继市场这一事实，接下来激励，防攻击，中继与矿工关系等等一系列额外的复杂问题，让落地更成为不现实。为此本文提出了 State Providers 来统一费用市场减少系统的复杂度。只需将交易广播到 mempool，并添加必要的见证数据（来自 State Providers），State Providers 不需要创建交易包。它们已通过状态通道获得支付。

L2 作为 L1 执行的扩展层，执行的性能必然是重点关注和突破的点，UTXO 天然支持并行执行，所以很多 L2 的解决方案都选择了 UTXO 作为状态数据模型，自然对它的讨论也变得越来越多也更有价值了，接下来两片文章主要是针对 UTXO 进行的讨论。

[Practical parallel transaction validation without state lookups using Merkle accumulators](https://ethresear.ch/t/practical-parallel-transaction-validation-without-state-lookups-using-merkle-accumulators/5547)
本文提出了针对 UTXO 的一种不进行查找（或者说更高效的查找）的并行执行方案：
1. 每个交易要包含（更像是绑定一个 block 高度），然后在这个高度之后的 D 个block 内必须执行，否则交易验证不通过。
2. UTXO 只需要最多向前查询 D 个 block 就可以知道是否被花费。
3. 每个 block 的状态数据，份两部分删除的 UTXO 和新增的 UTXO，用 SMT 实现，主要是 log(N)复杂度很稳定无平衡（但是 N 不是叶子节点数是空间数量）。
   1. 注意：这个UTXO 如果也不在 D 个 block 之前或者说在 D 个 block 内合法，但是确实违法的，提到的想法是有一个外围接受交易的节点来进行校验（同时提供绑定的块高）。
4. 除了最后统一更新状态增量的部分，其余都是可以完全并发的

[Compact Fraud Proofs for UTXO Chains Without Intermediate State Serialization](https://ethresear.ch/t/compact-fraud-proofs-for-utxo-chains-without-intermediate-state-serialization/5885?ref=fuel-labs.ghost.io)
本文提出了一个基于 UTXO 不需要中间根且可以并行验证欺诈证明的方案。区块头现在包含三个（3）数据根（SMT）：有序输入，有序输出，有序交易。 交易 ID 就是用户必须签名的哈希值，交易的输入不再引用一个未花费的输出，而是引用有序输入列表中的一个索引，输出则引用有序输出列表中的一个索引。输入树的叶子节点可以找到，UTXO 作为输出的信息。输出一样包含当前交易的信息。针对输入输出和交易可以分别提供欺诈证明。完全并行的，因为基于 UTXO。

## Summarize
本文介绍了很多社区讨论的方案草稿或者一些论文，这些内容其实都没有完整的落地项目，但是在已落地的项目中又都能看到他们的影子，所以本文根据数据可用性的理论结构介绍了这些方案草稿，方便理解 L2 数据可用性的理论结构以及社区讨论的热点以及方便理解后续文章讨论的点在哪一个理论结构位置，对后续深入理解 L2 大有裨益。

本文除了介绍一些奠基性的方案草稿之外，最重要的意义是梳理了 L2 数据可用性的理论结构（包含的内容，内在关系等），可以帮助建立理论体系。

后续会介绍 Op Rollup 的具体实现，结合这部分内容会更深刻的理解：理论设计与现实落地的距离以及为什么，也会理解理论结构的设计原因等等。


## Reference
1. [Validity Proofs vs. Fraud Proofs](https://medium.com/starkware/validity-proofs-vs-fraud-proofs-4ef8b4d3d87a)
2. [Fraud and Data Availability Proofs: Maximising
   Light Client Security and Scaling Blockchains
   with Dishonest Majorities](https://arxiv.org/pdf/1809.09044)
3. [A note on data availability and erasure coding](https://github.com/ethereum/research/wiki/A-note-on-data-availability-and-erasure-coding)
4. [A data availability blockchain with sub-linear full block validation](https://ethresear.ch/t/a-data-availability-blockchain-with-sub-linear-full-block-validation/5503)
5. [Cryptographic proof of custody for incentivized file-sharing](https://distributedlab.com/whitepaper/ProofOfcustody.pdf)
6. [Towards on-chain non-interactive data availability proofs](https://ethresear.ch/t/towards-on-chain-non-interactive-data-availability-proofs/4602)
7. [On-Chain Non-Interactive Data Availability Proofs](https://ethresear.ch/t/on-chain-non-interactive-data-availability-proofs/5715?ref=fuel-labs.ghost.io)
8. [Data availability proof-friendly state tree transitions](https://ethresear.ch/t/data-availability-proof-friendly-state-tree-transitions/1453)
9. [Practical parallel transaction validation without state lookups using Merkle accumulators](https://ethresear.ch/t/practical-parallel-transaction-validation-without-state-lookups-using-merkle-accumulators/5547)
10. [The Stateless Client Concept](https://ethresear.ch/t/the-stateless-client-concept/172)
11. [Compact Fraud Proofs for UTXO Chains Without Intermediate State Serialization](https://ethresear.ch/t/compact-fraud-proofs-for-utxo-chains-without-intermediate-state-serialization/5885?ref=fuel-labs.ghost.io)
12. [On-chain scaling to potentially ~500 tx/sec through mass tx validation](https://ethresear.ch/t/on-chain-scaling-to-potentially-500-tx-sec-through-mass-tx-validation/3477)
13. [Layer 2 state schemes](https://ethresear.ch/t/layer-2-state-schemes/5691)
14. [Multi-Threaded Data Availability On Eth 1](https://ethresear.ch/t/multi-threaded-data-availability-on-eth-1/5899?ref=fuel-labs.ghost.io)
15. [State Providers, Relayers - Bring Back the Mempool](https://ethresear.ch/t/state-providers-relayers-bring-back-the-mempool/5647)
