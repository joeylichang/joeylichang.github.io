Ph1.a：Proposer 向所有的 Acceptor 发送 prepare 请求，请求中带有 proposalid（Proposalid 全局唯一，且应该是全局最大）。

Ph1.b：Acceptor 接收 prepare 请求，如果 prepare 带有的 proposalid 小于等于当前 Acceptor 的 proposealid 则拒绝请求，否则返回 accept 最大的 proposalid 及其 内容。



Ph2.a：Proposer 收集所有的 prepare 请求，收集多数 acctptor 的返回结果后，选择返回的最大的 proposalid 及其内容作为提案发起 accept 请求，如果内容为空，则使用 Proposer 自身的 proposalid 和内容作为提案。

Ph2.b：Acceptor 接收到 accept 请求，如果 proposalid 小于自身的 proposalid 则，拒绝请求，否则持久化待 learner 学习，然后返回成功。 Proposer 收集 accept 的结果，如果多数返回成功，则提案提交成功。



原则：后者认同前者（分为前者是否成功确认提案的两种情况）

1. 确定之前的 proposalid 无法确定提案时（所有的 prepare 请求中最大的 proposalid 对应的内容为空，可以认为之前的提案已经确定并处理过了，或者还没有提案），新的 proposalid 提交自己的内容，不会冲突。
2. 一旦之前的 proposalid 确定了取值（所有的 prepare 请求中最大的 proposalid 对应的内容不为空），新的 proposalid 认同它，不会破坏它。



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

    

    ** 协议中没有介绍 learner 的逻辑，这其实更偏向于实现，需要在后面继续看，而后回来重新审视上述问题，带着问题去看 PhxPoxos ，被少数 Acceptor 接受的提案 被 learn 了 如何纠正的**

    