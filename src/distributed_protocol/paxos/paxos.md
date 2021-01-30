Ph1.a：Proposer 向所有的 Acceptor 发送 prepare 请求，请求中带有 proposalid。

Ph1.b：Acceptor 接收 prepare 请求，如果 prepare 带有的 proposalid 小于等于当前 Acceptor 的 proposealid 则拒绝请求，否则返回 accept 最大的 proposalid 及其 内容。



Ph2.a：Proposer 收集所有的 prepare 请求，收集多数 Acceptor 的返回结果后，选择返回的最大的 proposalid 及其内容作为提案发起 accept 请求，如果内容为空，则使用 Proposer 自身的 proposalid 和内容作为提案。

Ph2.b：Acceptor 接收到 accept 请求，如果 proposalid 小于自身的 proposalid 则，拒绝请求，否则持久化待 Learner 学习，然后返回成功。 Proposer 收集 accept 的结果，如果多数返回成功，则提案提交成功。



上述内容介绍了 Paxos 协议的核心内容，也是协议的精髓。但是在理解层面上还是有疑惑：

1. Acceptor 接收 Proposer 的 accept 请求之后，何时 Learner 去学习它？
2. 何时 Acceptor 会将学习过的内容清空，便于下一轮提案的正确进行（如果 Learner 学习过了，但是 Acceptor 没删除，Proposer 发起了下一轮的 prepare 请求，让然处理的是未删除的内容，无法开启新的提案）？
3. 等等……

上述理解上的困惑其实都是来自于学习过程中”脑图方法“需要有一个具体的模型来辅助我们理解。而 Paxos 论文更像是一个抽象的理论，加之 Lamport 晦涩的措词 和 没有具体的实现伪码（与 raft 相比）造成了这些问题。

下面对前一部分介绍的 Paxos 核心协议，做一个 Learner 部分的补充，方便我们后续的理解，这个 Learner 部分的补充是结合 PhxPaxos 的实现。

Ph3.a：Proposer 收集 accept 的返回结果，如果得到多数返回成功。Proposer 向所有的 Learner 发送 learn 请求学习提案内容。

Ph3.b：Learner 接收到 learn 请求之后，更新本地的状态机，并删除 Acceptor（与 Learner 同进程）本轮提案的内容。



原则：后者认同前者（分为前者是否成功确认提案的两种情况）

1. 确定之前的 proposalid 无法确定提案时（所有的 prepare 请求中最大的 proposalid 对应的内容为空，可以认为之前的提案已经确定并处理过了，或者还没有提案），新的 proposalid 提交自己的内容，不会冲突。
2. 一旦之前的 proposalid 确定了取值（所有的 prepare 请求中最大的 proposalid 对应的内容不为空），新的 proposalid 认同它，不会破坏它。



Multi-Paxos

前面介绍的数朴素 Paxos，朴素 Paxos 的核心部分每一轮提案都需要 prepare / accept 请求



1. 在什么情况下可以认为任务提案被确定，不再更改？

   Ph2.a 之后受到多数 Acceptor 的正确返回。

   如果少数 Acceptor 返回正确：

   ​	在下一轮的提案中 Ph1.a 收集多数 prepare 的返回时，

   ​	可能恰好收集的多数结果没有包含之前被少数 Accetpor 接受的提案，

   ​	此时该提案没有被多数 Acceptor 提交，相当于提交失败。

   如果少数 Acceptor 返回正确了，其他 Acceptor 有网络延时，要分情况讨论：

   1. 已经发起了下一轮的 prepare，恰好收集多数的结果时没有包含接受上一轮的提案，此时情况如上（接受的持久化写入状态机也无所谓没有达到多数，也读取不到）。

    	2. 已经发起了下一轮的 prepare，此时部分延时 Acceptor 接受了提案，此时 prepare 一定会返回该提案（一定收集多数的结果），下一轮的 prepare 遵从”后者认同前者原则“，会继续提交该提案。
    	3. 已经发起了下一轮的 prepare，此时收集到了少数接受提案的 Acceptor 的返回值，则会重新提交该提案。

2. Paxos 的两个阶段分别在做什么？

   Ph1 通过 proposalid 抢占集群资源，并确认提案内容。

   Ph2 提交提案，达到半数以上认为提交成功。

3. 一个 proposalid 会有多个 Proposer 进入第二阶段吗？

   不会，因为在 Ph1.a 中 Acceptor 会拒绝小于等于当前 proposalid 的请求，并且 Proposer 需要收集多数的结果，相同 proposalid 的 Proposer 只会选出一个。

4. 在什么情况下，Proposer 可以将提案设置为自身的提案内容？

   收到多数 Acceptor 的 prepare 请求中最大的 proposalid 的内容为空时，使用资深的内容作为提案。

   此时，要么是被 learner 学习完删除（要看具体的实现，后面会结合 PhxPaxos） 或者 之前的提案没有提交完。

5. 在第二阶段（Ph1.b），如果获取的提案内容为空，为什么可以保证旧的 proposalid 无法形成确定性取值？

   收集多数的 prepare 结果，如果有确定性取值的话一定可以获取到，否则不一定取到。

   如果少数 Acceptor 接受了，此时 Proposer 的 prepare 返回结果中没有，

   可能会被覆盖 或者 会被写入状态机（需要结合 PhxPaxos） ，及时写入状态机也读不到，因为读取也是需要读多数。

6. 新的 proposalid 获取成功，旧 proposalid 的 Proposer 如何运行？

   旧 proposalid 处于 Ph2 阶段，其中 Ph2.a 会正常继续发送 accept 请求，但是 Accept 会拒绝小于当前 proposalid 的请求。

7. 如何保证新的 proposalid 不破坏已经达成的确定性提案？

   如上所述 ”后者认同前者“ 原则，体现在 Ph1.a 阶段。

8. 为什么在第二阶段（Ph1.b）存储提案内容时，只需要考虑 proposalid 最大的取值？

   因为 Ph1.a 阶段需要收集多数 prepare 的结果，proposalid 最大的就是最近的上一轮，比这个小的一定是更之前的轮次。

   每个轮次都是会收集多数 prepare 的结果，类似数学归纳法，在具体实现中如果差距较大会使用 checkpoint 进行数据同步。

9. 在形成确定性提案之后出现任意半数以下的 Acceptor 故障，为何确定的提案不会被更改？

   因为下一个轮次在收集 prepare 请求结果时，需要收集半数以上，一会不会遗漏已经提交成功的提案。

10. 如果 Proposer 运行过程中，半数以下的 Acceptor 故障，此时将如何运行？

    按上述协议正常运行，不会影响结果。

11. 正在运行的 Proposer 和 半数以下的 Acceptor 同时故障，提案的内容可能是什么情况？为何之后新的 Proposer 可以形成确定性提案？

    Ph1.a 阶段发生故障，此时新的 Proposer 可以继续下一轮的 proposalid 协商，因为 Ph1.b 中会接受大于当前 proposalid 的心 prepare 请求。

    Ph1.b 故障的 Acceptor 无法返回结果，但是不影响协议继续，因为 Proposer 只需要半数以上的结果即可。

    Ph2 阶段发生故障，没有发送accept请求如上；发送被少数接受，该提案可能被下一轮提交，可能不会被提交；发送被多数Acceptor接受，一定会被提交。

    新的 Proposer 在 Ph1.a 会收集多数的 prepare 结果，在 Ph1.b 会遵从”后者认同前者“ 原则。

    

    **协议中没有介绍 learner 的逻辑，这其实更偏向于实现，需要在后面继续看，而后回来重新审视上述问题，带着问题去看 PhxPoxos ，被少数 Acceptor 接受的提案 被 learn 了 如何纠正的**



PhxPaxos 如何解决活锁

​	控制提案提交的频率

  1. 首先，在一个 Proposer 内部提案一定是串行的，PhxPaxos 使用自研锁控制并发。

  2. 其次，要控制提交的频率避免活锁，正式在自研锁部分实现的提交频率控制：

       1. 等待锁的提案不能过长，既等待队列长度（默认没有设置）。

       2. 等待时间（默认没有设置）：

            		1. 平均等待锁的时间超过设置的等待时间，m_iRejectRate 增长一个步长（默认值 3）；
            		2. 否则，m_iRejectRate 减去一个步长（默认值 3）；
            		3. 引入随机性，判定是否可以获取锁，随机性与 m_iRejectRate 负相关，既越大 获取的概率越低；
            		4. 平均等待时间，没间隔 250次 统计一次。

     		3. 通过上述控制逻辑之后，才允许排队获取锁资源。

          		1. 默认锁的超时时间是 1s，然后无限重试；

               

		3. 最后，审视一下这个策略：

     		1. 反馈迟缓：m_iRejectRate = 0 时，系统状态不好，等待锁的时间较长，步长为 3 时 m_iRejectRate 增长较慢（间隔 250次只能增长 3），当前请求通过的概率仍较大，反之下降也较慢；
     		2. 是否应该考虑等待加锁的队列长度？

     

​	重试增加带有避让时间：这部分需要客户端完成



Mutl-Paxos 核心思想



PhxPaxos 实现

1. Proposer 发起提案之前的check：CheckNewValue

   1. m_oCommitCtx.IsNewCommit()

      m_oCommitCtx 可以认为是客户端在服务端的抽象，它发起了提案，

      如果其上一个提案没有结束，不能进行新的提案

   2. Learner.InstanceID + 1 >= Learner.m_llHighestSeenInstanceID

      用来判断 learner 是否学习了最新的数据，否则不能开始新的提案

      **这块不是很理解，需要在后面一个提案结束之后一起看一下**

   3. m_oCommitCtx.StartCommit(m_oProposer.GetInstanceID());

      给提案设置 InstanceID，应该是最新的 InstanceID，既前一轮结束之后 ++ 的结果

      这里要多说一点，在 muti-paxos 中：

      1. proposalid 类似 gossip 中的 configepoch 
      2. InstanceID 类似 gossip 中的 currentepoch
      3. 每次提案 InstanceID++，只有 主变化时，发起 prepare 请求才更新 proposalid++
      4. 区别是 proposalid 与 InstanceID 无关 完全是两套单独计数

   4. m_oProposer.NewValue 开始发起 prepare 请求

2. Proposer 发送 prepare 请求：m_oProposer.NewValue/Prepare

   1. 如果 m_oProposerState.GetValue() 为空，设置用户本次提案的内容（也不是最终内容）
      1. 如果 m_oProposerState 不为空表示之前的提案没有结束，m_oCommitCtx 可能因为异常或者超时被清空了，此时需要继续提交之前的数据
   2. 设置 prepare 请求参数，然后发送 prepare 请求，有两个参数需要注意 
      1. InstanceID，既 提案的编号，如前所述是前一个天结束之后 ++
      2. proposalid，通过一个标志（bool）决定是否 ++
      3. 当有请求被拒绝、超时等情况会更新这个标志，进而完成proposalid 更新，最后开始重新选主

3. Acceptor 接收 prepare 请求

   1. 加速 learn 逻辑

      **没理解，可能需要后面learn的逻辑补充**

   2. prepare 请求中的 InstanceID == Acceptor 的 InstanceID 表示一个轮次

      1. prepare 请求

         1. 进行投票，有一些需要说明

            Acceptor 内部会记录 Promise ，既他之前选的主，这里会比较 prepare 请求中的 proposalid 与 Promise 的 proposalid，大于等于会投票（注意比较的不是 instanceID， instanceID 在前面校验过了）

         2. Acceptor 会判断当前提案是否被学习并清空，否则会把内容返回去

         3. 处理协议中要求的信息，还会返回一些 proposalid、instanceid 等用于让其他节点看到全局的最大值，方便后面重新选举和优化

      2. accept 请求

         1. 见下面 6 部分

   3. prepare 请求中的 InstanceID > Acceptor 的 InstanceID 表示中间有请求流失，会加入队列等待前面的请求达到然后按顺序执行

   4. repare 请求中的 InstanceID < Acceptor 的 InstanceID，说明这个 InstanceID 的内容已经处理过了，这个请求可能是网络延时导致的，直接忽略即可

4. Proposer 收集 prepare 请求结果

   1. 比较 proposalid 是否相等，既一个主发出去的请求
   2. 统计投票结果
      1. 超过半数直接发起 accept 请求
      2. 否则，发起重试，需要设置一些变量，例如：上述标志位，在写一次 prepare 需要提升 proposalid 等
      3. 同时会更新一些，全局的信息如上所述：新的 promise-proposalid、instance 等等

5. Proposer 发起 accept 请求

   1. 如协议，根据 prepare 返回结果确定提案内容 发起 accept 请求
   2. 需要注意的是，一旦 accept 失败将会从 prepare 重新开始

6. Acceptor 接收 accept 请求

   1. 投票逻辑  与 接收 prepare 基本一致
   2. 需要注意的是，这两部分信息都需要落盘用于重启

7. Proposer 收集 accept 请求结果

   1. 逻辑与收集 prepare 请求结果相同
   2. 区别是收到半数以上的 accept 向 learner 发起学习

8. Learner 开始学习

   1. 主 Proposer 向所有的 learner 发起了学习请求
   2. Learner 会校验 InstanceID 是否代表一个轮次
   3. Learner 会从 Acceptor 中获取投票的 promise 是否与 发起的Proposer 一致
   4. 交给状态机执行

9. 结束本轮提案

   1. 每个节点执行之后会执行 NewInstance();
      1. Proposer、Acceptor、Learn中的InstanceID++
      2. AcceptorState、ProposerState、LearnerState中的数据、投票信息都会清空
      3. 所以理想情况下Proposer、Acceptor、Learn中的InstanceID应该是一致的















































