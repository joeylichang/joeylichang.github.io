Ph1.a：Proposer 向所有的 Acceptor 发送 prepare 请求，请求中带有 proposalid（Proposalid 全局唯一，且应该是全局最大）。

Ph1.b：Acceptor 接收 prepare 请求，如果 prepare 带有的 proposalid 小于等于当前 Acceptor 的 proposealid 则拒绝请求，否则返回 accept 最大的 proposalid 及其 内容。



Ph2.a：Proposer 收集所有的 prepare 请求，收集多数 acctptor 的返回结果后，选择返回的最大的 proposalid 及其内容作为提案发起 accept 请求，如果内容为空，则使用 Proposer 自身的 proposalid 和内容作为提案。

Ph2.b：Acceptor 接收到 accept 请求，如果 proposalid 小于自身的 proposalid 则，拒绝请求，否则持久化待 learner 学习，然后返回成功。 Proposer 收集 accept 的结果，如果多数返回成功，则提案提交成功。



原则：后者认同前者（分为前者是否成功确认提案的两种情况）

1. 确定之前的 proposalid 无法确定提案时（所有的 prepare 请求中最大的 proposalid 对应的内容为空，可以认为之前的提案已经确定并处理过了，或者还没有提案），新的 proposalid 提交自己的内容，不会冲突。
2. 一旦之前的 proposalid 确定了取值（所有的 prepare 请求中最大的 proposalid 对应的内容不为空），新的 proposalid 认同它，不会破坏它。



1. 在什么情况下可以任务提案被确定，不再更改？
2. Paxos 的连个阶段分别在做什么？
3. 一个 proposalid 会有多个 Proposer 进入第二阶段吗？
4. 在什么情况下，Proposer 可以将提案设置为自身的提案内容？
5. 在第二阶段，如果获取的提案内容为空，为什么可以保证旧的 proposalid 无法形成确定性取值？
6. 新的 proposalid 全模组就按么成交额没好好拍卖行做健脾，旧 proposalid 的 Proposer 如何运行？
7. 如何保证新的 proposalid 不破坏已经达成的确定性提案？
8. 为什么在第二阶段存储提案内容时，只需要考虑 proposalid 最大的取值？
9. 在形成确定性提案之后出现任意半数以下的 Acceptor 故障，为何确定的提案不会被更改？
10. 如果 Proposer 运行过程中，半数以下的 Acceptor 故障，此时将如何运行？
11. 正在运行的 Proposer 和 半数以下的 Acceptor 同时故障，提案的内容可能是什么情况？为何之后新的 Proposer 可以形成确定性提案？