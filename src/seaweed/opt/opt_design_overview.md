### 目标

OS（ObjectStore）常见的业务场景：图片存储（微信、微博的图片）、视频存储（爱奇艺等视频网站）、短视频、小文件等等。这些场景共同的特点是数据量大，相对 QPS 不算高。

随着业务发展的不同阶段对系统的要求也是不一样的，例如百度个人网盘，随着业务发展数据成指数级增长，加之竞争对手的退出，机器成本变成敏感因素。再比如搜索、抓取类业务需要定时批量更新，需要集群能够承受一定的压力（虽然业务可以限流，但是要保证全量数据更新的周期——一般是天级别，仍然需要 OS 具备一定的性能）。相反业务流量不会很大，一般图片、热点视频都有 CDN 抗流量，回源比例可以控制在 5% 以下。批量灌库很可能伴有批量删库的场景，需要 OS 可以避免存储空洞来节省成本。

结合 seaweedfs 和 业务需要，对 seaweedfs 优化设定了如下两个目标：

1. 支持大规模集群：1k - 5k，总容量 > 3PB。
2. 支持较大流量：性能随机器数量线性增长。



### 原则

seaweedfs 目前社区比较活跃，还有很多发展空间，为了保持架构上的优化不与社区源码产生较大差异而无法进行 rebase（吸收社区有价值的 feature），尽量避免对源码有较大侵入。



### 架构图

<img src="../../../images/seaweed_arch_opt.png" alt="seaweed_arch_opt" style="zoom:50%;" />

在之前的[文章](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/overview.md)中已经介绍过上述优化架构各模块的主要功能，不再赘述，后续有文章介绍详细设计。

在这里需要说两点：

1. Alloc-Cluster 和 VolumeService 除了分摊 Master 压力，避免 Master 性能成为集群瓶颈之外，还实现了集群性能随机器数量线性变化的目的。

2. Repair-Cluster 是社区版自愈能力不足的一个补充，同时独立一个集群也做到了对源码最小的侵入，保证了 Master 、 VolumeServer 后期的扩展性。

   

### 约束

Seaweedfs 支持 collection 逻辑概念，强制 collection 与业务一一对应，这样做的好处是可以在 volume 级别对业务进行隔离。随着业务的发展和业务优先级，会有搭建独立集群的需求，强制 collection 与业务一一绑定可以更方便实现业务从混部集群剥离。

封禁通过 Master 中转的接口，例如：submit，其设计之初的目的是较少业务端与集群之间的请求次数，缺点是如果业务过分使用会吃掉 Master 很多系统资源，比如 CPU 、网络带宽等。考虑业务在使用接口的不可控 和 系统运维必要的可控性，对需要 Master 中专的接口进行封禁。



### 优化

##### 写接口

 对 Assgin 接口进行优化，社区版本中 Master 的结果返回主 VolumeServer 的地址，写主成功之后查询 Master 获取从 VolumeServer 地址，再同步写成功之后返回业务成功。设计的初衷是，在写期间可能 volume 路由发生变化，在主 VolumeServer 写成功之后，可以避免因路由变更引起的失败。

机器故障、Rebalance 从一个较长的周期来看不会是高频时间，加上机器规模在千台以上，故障机器的volume 占比也不是较大， 主 VolumeServer 需要重新请求 Master 获取路由，不仅耗时、而且对Master 造成一定压力，同时大部分可能都是浪费的。

Assgin 请求 Master 之后，增加一个从Master数组的字段将副本地址一并返回，主Volume 写完本地之后直接向从 Volume 发起写请求。

##### Compact

社区版 Compact 存在以下几个问题：

1. 可控性不强：系统或者外围监控不能监控目前进度，没有办法做全局协调。
2. 异常处理不完善：在commit阶段失败，会造成空间浪费没有处理机制。

结合上述问题，对 Compact 进行优化：

1. Master 侧维护全局 compact 队列，对 compact 任务进行抽象，好处是可以通过全局队列控制全局的compact 流量，可以通过接口查询任务执行程度。
2. VolumeServer 侧维护 Compact 任务及其状态机，可以通过cli（交互式命令行运维工具）完成任务详细情况查询和控制，可以设置一个节点只能并发执行一个volume 的 compact。
3. commit阶段故障可以在 volumeServer 侧及时告警，通过 cli 清理多余的文件。
4. 对于 Compact 流程还增加了只读阶段（设计的思考在 Repair 部分介绍）。
