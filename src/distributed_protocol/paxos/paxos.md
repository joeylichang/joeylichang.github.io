# Paxos

## 协议内容

Ph1.a：Proposer 向所有的 Acceptor 发送 prepare 请求，请求中带有 proposalid。

Ph1.b：Acceptor 接收 prepare 请求，如果 prepare 带有的 proposalid 小于等于当前 Acceptor 的 proposealid 则拒绝请求，否则返回 accept 最大的 proposalid 及其 内容。



Ph2.a：Proposer 收集所有的 prepare 请求，收集多数 Acceptor 的返回结果后，选择返回的最大的 proposalid 及其内容作为提案发起 accept 请求，如果内容为空，则使用 Proposer 自身的 proposalid 和内容作为提案。

Ph2.b：Acceptor 接收到 accept 请求，如果 proposalid 小于自身的 proposalid 则，拒绝请求，否则持久化待 Learner 学习，然后返回成功。 Proposer 收集 accept 的结果，如果多数返回成功，则提案提交成功。



## 协议补充

上述内容介绍了 Paxos 协议的核心内容，也是协议的精髓。但是在理解层面上还是有疑惑：

1. Acceptor 接收 Proposer 的 accept 请求之后，何时 Learner 去学习它？
2. 何时 Acceptor 会将学习过的内容清空，便于下一轮提案的正确进行（如果 Learner 学习过了，但是 Acceptor 没删除，Proposer 发起了下一轮的 prepare 请求，让然处理的是未删除的内容，无法开启新的提案）？
3. 等等……

上述理解上的困惑其实都是来自于学习过程中”脑图方法“需要有一个具体的模型来辅助我们理解。而 Paxos 论文更像是一个抽象的理论，加之 Lamport 晦涩的措词 和 没有具体的实现伪码（与 raft 相比）造成了这些问题。

下面对前一部分介绍的 Paxos 核心协议，做一个 Learner 部分的补充，方便我们后续的理解，这个 Learner 部分的补充是结合 PhxPaxos 的实现。

Ph3.a：Proposer 收集 accept 的返回结果，如果得到多数返回成功。Proposer 向所有的 Learner 发送 learn 请求学习提案内容。

Ph3.b：Learner 接收到 learn 请求之后，更新本地的状态机，并删除 Acceptor（与 Learner 同进程）本轮提案的内容。



## 协议基础

原则：后者认同前者（分为前者是否成功确认提案的两种情况）

1. 确定之前的 proposalid 无法确定提案时（所有的 prepare 请求中最大的 proposalid 对应的内容为空，可以认为之前的提案已经确定并处理过了，或者还没有提案），新的 proposalid 提交自己的内容，不会冲突。
2. 一旦之前的 proposalid 确定了取值（所有的 prepare 请求中最大的 proposalid 对应的内容不为空），新的 proposalid 认同它，不会破坏它。



## Multi-Paxos

前面介绍的是朴素 Paxos，朴素 Paxos 的核心部分每一轮提案都需要 prepare / accept 请求，如果每次发起 prepare 请求的 Proposer 都是同一个的话，可以省略后面的 prepare 请求，既一个 prepare + n 个 accept，减少 n - 1 次的网络请求在性能上将会有很大的提升——Multi-Paxos。

后面会结合 PhxPaxos 详细介绍一种 Multi-Paxos 的实现，在此需要详细说一下 ”如果每次发起 prepare 请求的 Proposer 都是同一个“ 的含义，在很多资料中称作主，指定一个 Proposer 也被称作选主，其实 Paxos 论文中没有这个概念，这里的 Proposer 也不是常规理解的主，因为任何一个 Proposer 都可以通过提升 proposalid 来更改这个主，并且 Paxos 协议也没有选主的逻辑。

但是，这个特殊的 Proposer 在一段时间内只有它负责提案又像是传统意义上的主，暂且称为”狭义上的主“吧，后面的介绍也会使用”主“这个词，但是我们应该意识到和其它强一致算法选出的主是不一样的（同一时刻不一定只有一个、协议没有选主的逻辑，任何一个点都可以是主、也没有其它强一致算法主与从之间的强制约束等等）。



## PhxPaxos

### 基础

在正式介绍 PhxPaxos 实现 Multi-Paxos 协议之前，需要对一些基础知识进行梳理，方便后面的理解。

Proposalid vs. InstanceID

正如前述 Multi-Paxos 协议，只有在切换提交的 Proposer 时才提升 Proposalid，大部分时间 Proposer 固定，此时 Proposalid 不提升保持一致，可以理解为 Proposalid 标志 Proposer 的切换。每轮提案的提交试用的标识是 InstanceID，既一个 Proposalid 代表一段时间内固定的 Proposer，对应若干个 InstanceID。

Proposer && Acceptor && Learner

PhxPaxos 中有三个对象：Proposer、Acceptor、Learner 分别对应 Multi-Paxos 协议中相应的角色。在 PhxPaxos 实现中这三个角色在一个进程中，且共用一个线程。Proposer、Acceptor、Learner 三个类都继承自一个基类 Base，Base 基类中有一个 InstanceID 私有变量，每一轮提案结束之后，三个对象的 InstanceID 都会 ++，既每一个时刻三个角色的 InstanceID 都是相同的，这也是模块交互期间标识是否是同一轮次提案的依据。

Proposer、Acceptor、Learner 三个对象都各自的 state 私有变量（既 ProposerState、AcceptorState、LearnerState）分别代表各自的环境变量（类似其他代码中 context 的概念），在每一轮提案 Learn 完毕之后，除了 InstanceID++，还会清空这三个模块的 state 变量，这样在下一轮 prepare 时 Acceptor 会返回空，否则遇到异常等情况不会清空，在下一轮提案会继续提交该内容。



### 实现

1. Ph1.a ：Proposer 发起 prepare 请求

   1. InstanceID++，ProposerState 记录的提案内容不为空使用其作为暂时提案内容（Accept 收集阶段还会改），否则使用业务的内容。

      先去 check ProposerState 中记录的内容，可能是因为超时等异常情况导致之前的提案没有结束需要重试等情况。

   2. 发送 prepare 请求，其中带有 InstanceID && proposalid。

      1. 其中 proposalid，通过一个标志（bool）决定是否 ++。
      2. 当有请求被拒绝、超时等情况会更新这个标志，开始重新执行 prepare，否则复用 proposalid。

2. Ph1.b：Acceptor 响应 prepare 请求

   1. 校验 InstanceID：Proposer.InstanceID == Acceptor.InstanceID
   2. 进行投票，投票先比较 proposalid，相等之后比较 nodeid（Acceptor 会记录之前 Promise 的信息）
   3. **没有特殊情况（特殊情况通过标志位表示，请求失败、投票被拒绝等），直接跳入 3.3 发送 accept 请求**

3. Ph2.a：Proposer 收集 prepare 请求结果，并发送 accept 请求

   1. 校验：proposalid 是否一致，判断是否是自己发出的请求。
   2. 统计投票结果，如果有票被拒绝，会更新 1.2.2 中的标志位，下一轮提案需要从 prepare 开始。
   3. 超过半数投票，则发起 accept 请求
      1. 选取 InstanceID 最大的内容为提案内容（如前介绍没有 learn ），如果为空使用 ProposerState 记录的内容。
      2. 如果失败（超时等异常情况），需要从 prepare 阶段重试。

4. Ph2.b：Acceptor 响应 accept 请求

   1. 投票逻辑 与 2.2 相同。
   2. Acceptor 对于 prepare 和 accept 提案的内容必须落盘（用于重启）。

5. Ph3.a：Proposer 收集 accept 请求，并发送 learn 请求

   1. 收集投票逻辑 与 4.1 相同
   2. 发送 learn 请求：Proposer 会向所有的 Learner 发送 learn 请求。

6. Ph3.b：Learn 响应 learn 请求

   1. Learner 校验 InstanceID，并且 Learner 从 Acceptor 回去 Promise 信息进行节点校验。
   2. 交给状态机执行。
   3. 结束本轮提案：
      1. Proposer、Acceptor、Learn中的InstanceID++
      2. AcceptorState、ProposerState、LearnerState中的数据、投票信息都会清空。

上述只是只是针对 PhxPaxos 对 Multi-Paxos 协议核心部分的实现进行了概要介绍，详情见：

1. [PhxPaxos分析之提案控制策略篇](https://blog.csdn.net/weixin_41713182/article/details/88213176)

2. [PhxPaxos分析之协议实现篇](https://blog.csdn.net/weixin_41713182/article/details/88304530)

3. [PhxPaxos分析之Learner 篇](https://blog.csdn.net/weixin_41713182/article/details/88355009)

   

整体上结合协议内容阅读代码难度不大，需要注意 InstanceID 与 Proposalid 的含义及如何使用。以及 完整的提案周期内 Proposeor、Acceptor、Learner 之间怎么用 InstanceID 串联起来 和 相关环境数据被清空的便于下一轮提案得以继续进行，这些在 Paxos 论文中都是没有描述的具体实现。

除此之外 PhxPaxos 还进行了一下工程优化：

1. 



### 活锁

多个Proposer同时发起Prepare请求，导致没有一个Proposer获得多数派的投票进而重试，循环往复的现象称为活锁。最后，介绍一些 PhxPaxos 是如何解决活锁问题的，其实解决活锁常见的有两种方式：控制提案提交的频率、重试增加带有避让时间。PhxPaxos 具体实现如下：

1. 在一个 Proposer 内部提案一定是串行的，PhxPaxos 使用自研锁控制并发。

2. 在自研锁内部实现提交频率 和 随机时间重试（默认不开启）。
   1. 等待锁的提案不能过长，既等待队列长度（默认没有设置）。
   2. 等待时间（默认没有设置）：
      1. 平均等待锁的时间超过设置的等待时间，m_iRejectRate 增长一个步长（默认值 3）；
      2. 否则，m_iRejectRate 减去一个步长（默认值 3）；
      3. 引入随机性，判定是否可以获取锁，随机性与 m_iRejectRate 负相关，既越大 获取的概率越低；
      4. 平均等待时间，没间隔 250次 统计一次。
   3. 通过上述控制逻辑之后，才允许排队获取锁资源。
   
3. 默认情况第 2 步中的队列长度 和 等待时间都没有设置，而是无限重试间隔 1s。

   

## 深入理解

1. 在什么情况下可以认为任务提案被确定，不再更改？

   多数节点 accept 之后。少数节点 accept 的节点，通过 InstanceID 判断，当前请求比当前 Acceptor 处理的提案快若干个轮次，此时回将请求加入重试队列（IO 处理任务的队列），等待这期间 gap 的 InstanceID 请求到达按顺序处理。 

2. Paxos 的两个阶段分别在做什么？

   Ph1 通过 proposalid 抢占集群资源，并确认提案内容。

   Ph2 提交提案，达到半数以上认为提交成功。

3. 一个 proposalid 会有多个 Proposer 进入第二阶段吗？

   不会，因为在 Ph1.a 中 Acceptor 会拒绝小于等于当前 proposalid 的请求，并且 Proposer 需要收集多数的结果，相同 proposalid 的 Proposer 只会选出一个。

4. 在什么情况下，Proposer 可以将提案设置为自身的提案内容？

   收到多数 Acceptor 的 prepare 请求中最大的 proposalid 的内容为空时，使用自身的内容作为提案。

   大多数 Acceptor 上的 proposalid 只会相差一个，相差过多的节点不会超过半数（数学归纳法）。

   如果有 Acceptor 之间 proposalid 相差大于 1，可能是网络异常等因素导致，会有补偿机制（将超前的请求加入IO 处理任务的队列 && Learner 周期对比学习进度等补偿机制）。

5. 在第二阶段（Ph1.b），如果获取的提案内容为空，为什么可以保证旧的 proposalid 无法形成确定性取值？

   旧提案在发送 prepare 或者 accept 请求时，通过 InstanceID 判断轮次拒绝旧的提案相关请求。

6. 新的 proposalid 获取成功，旧 proposalid 的 Proposer 如何运行？

   仍继续运行，但是接收请求的 Acceptor 会通过 proposalid 拒绝旧请求。

7. 如何保证新的 proposalid 不破坏已经达成的确定性提案？

   如上所述 ”后者认同前者“ 原则，体现在 Ph1.a 阶段。

8. 为什么在第二阶段（Ph1.b）存储提案内容时，只需要考虑 proposalid 最大的取值？

   因为 Ph1.a 阶段需要收集多数 prepare 的结果，proposalid 最大的就是最近的上一轮，比这个小的一定是更之前的轮次。每个轮次都是会收集多数 prepare 的结果，数学归纳法，可以证明最大的是最近一轮且被多数接受的。

9. 在形成确定性提案之后出现任意半数以下的 Acceptor 故障，为何确定的提案不会被更改？

   因为下一个轮次在收集 prepare 请求结果时，需要收集半数以上，一会不会遗漏已经提交成功的提案。

10. 如果 Proposer 运行过程中，半数以下的 Acceptor 故障，此时将如何运行？

    按上述协议正常运行，不会影响结果。

11. 正在运行的 Proposer 和 半数以下的 Acceptor 同时故障，提案的内容可能是什么情况？为何之后新的 Proposer 可以形成确定性提案？

    Ph1.a 阶段发生故障，此时新的 Proposer 可以继续下一轮的 proposalid 协商，因为 Ph1.b 中会接受大于当前 proposalid 的心 prepare 请求。

    Ph1.b 故障的 Acceptor 无法返回结果，但是不影响协议继续，因为 Proposer 只需要半数以上的结果即可。

    Ph2 阶段发生故障，没有发送accept请求如上；发送被少数接受，该提案可能被下一轮提交，可能不会被提交；发送被多数Acceptor接受，一定会被提交。

    新的 Proposer 在 Ph1.a 会收集多数的 prepare 结果，在 Ph1.b 会遵从”后者认同前者“ 原则。

