# Min.IO

### 背景

Min.IO 是由 GlusterFS 班底（Anand Babu Periasamy, Harshavardhana等）发起并研发，在设计上能够看出一些相似的理念。Min.IO 在架构上更倾向极简的设计理念，将大规模集群拆分成小的互相独立的分布式集群，在小集群内部避免使用复杂的协调架构，从而分而治之的解决分布式问题。

Min.IO 支持很多种模式：单机模式、纠删码模式、分布式模式等等。本系列文章重点关注可扩展的分布式模式。Min.IO 从架构上看最显著的特点是去中心化，既没有负责元数据维护的模块，从用户角度看最显著的特点兼容 S3 程度非常高。



### 集群架构

![minio_arch](../../images/minio_arch.png)

如图所示，Min.IO 的分布式架构是一个去中心的架构，每个节点都是存储节点且对等。请求可以被任意的打到任何节点并完成相应的请求返回给客户端。一般会在集群和用户层之间加一层负载均衡的模块，保证集群内每个节点负责均衡，不会出现过热的节点

Min.IO 的元数据作为数据以对象的形式使用纠删码存储在集群内部，对象数据的读写都会使用分布式锁的方式先加锁再写入，然后再解除锁。分布式锁是 Min.IO 实现的 dsync（min.io/pkg 中的库） 特点是轻量快速易用。分布式锁解决了数据读写一致性的问题，详清见[dist_lock](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/dist_lock.md)。

当集群需要扩容时，会部署同样架构的上述一个小集群，但是两个小集群之间彼此是互通的（通过启动参数，两个小集群是知道各自的节点和拓扑结构的）。多个小集群理论上可以无限扩展，从而完成集群整体的无限扩展。



### Min.IO 特点

##### RS 编码

由于对象存储设计出发点是解决海量存储，甚至相对 KV 类型的存储属于冷存储，对性能不是过分敏感，随着系统和业务发展会演变为成本敏感型的服务，纠删码正式在性能和成本之间一个很好的折中方案。业界很对对象存储也是通过纠删码来节省成本的，例如：AWS S3、百度云（BOS）等。

Min.IO 与 AWS S3 一样使用 RS 编码（纠删码的一种），简单说将数据分成 n 块，根据参数设定额外生成 m 个校验块，一个纠删码分组就是 (n + m) 块磁盘，只要数据丢失损坏少于 m 块，则数据就可以完全恢复。例如：n = 8，m = 4 ，则相当于 1.5副本，通过调整 n 和 m 的值，根据业务和系统实际情况，平衡数据可靠性和成本。

纠删码相较于常见的副本策略就是编解码会消耗 CPU 资源，在性能上自然不如后者（结合对象存储性能不敏感的特点，比较适合），Min.IO 使用的是 [klauspost/reedsolomon](github.com/klauspost/reedsolomon) 库，其在 CPU 层做了汇编级别的优化。



##### Bitrot

位衰减指的是写入磁盘的二进制数据流与读出来的不一致，造成这种现象的原因有很多比如：电磁辐射、磁盘上磁点消失等等，Min.IO 在经过 RS 编码之后还会使用 HighwayHash256S（可配置）对写入磁盘的二进制进行签名，在数据修复或者读出是会利用签名进行验证。

Min.IO 的数据修复（后面介绍）分位两种模式，一种是正常模式分片状态进行校验（纠删码分组，对象级别），另一种是深度模式会使用 HighwayHash256S 进行校验。前者是系统的例行任务，后者必须通过 Http 接口开启（两者均可通过接口开启）。



##### 直接 IO

Min.IO 在设计上都是极简主义，在引擎层也是，一个桶一个目录，桶目录下面若干个对象目录，一个对象目录只有一个对象的数据及其元数据（大对象可能有多个数据文件）。在涉及到所有的所有的磁盘操作时，都是使用的直接 IO 来提升性能。直接 IO 的好处是避免了用户态和内核态的缓存，提升了性能。



##### 兼容 S3

Min.IO 在设计之初的目标之一（除了极简主义）就是高度兼容 AWS 的 S3 各项功能，为此 Min.IO 有一系列子系统去支持。既下面代码架构图右侧的大部分子系统。这里有两点需要说明：第一，Min.IO 的核心是 erasure系列（后面介绍），其本身并不复杂，但是为了兼容 S3 引入了若干子系统导致代码在接口层的处理逻辑较为繁琐（这也是阅读源码最困扰的地方，可以先行阅读 S3 官网对子系统有一些了解，会加速我们对源码的理解）。第二，Min.IO 的源码分为两大部分，既 cmd 和 pkg，前者是 Min.IO 系统的主要逻辑，后者是为支持其主逻辑中一些子系统单独开发的库（有的是为了兼容 S3，例如：lifecycle、Pliocy等，有的不是为了兼容 S3，例如：dsync等），这些库都是可以在其他项目中被引用的。下面看一下 Min.IO 为兼容 S3 而设计的主要子系统：

* [policy](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#policyconfig)

  基于资源的访问策略（Bucket、Object 级别），S3 的权限管理，分两部分，既 基于资源的访问策略（Bucket、Object 级别）、基于用户的访问策略（IAM子系统）。

* [notification](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#notificationconfig)

  S3 的通知机制，针对 Bucket 的一些关注事件进行监听，以及系统内部一些需要全部节点或者的事件（更新 bloomfilter），都需要 notification 子系统完成。

* [lifecycle](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#lifecycleconfig)

  Lifecycle 可以针对 Bucket 内部的 Object 进行生命周期配置.

* [objectLock](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#objectlockconfig)

  对象锁的是否开启的配置。S3 支持的对象是用户级别的锁，目的是支持对象的 worm，既一次写多读，只有用户解锁了才能删除，对象锁分位两种一种锁，一种是合法持有，前者又分位两种模式 Governance 和 Compliance（详情见上述链接）。

*  [versioning](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#versioningconfig)

  Versioning 支持对象保存多个版本，如果是删除对象，则插入一个删除标记在最新的版本中，并且可以随时恢复之前的任意版本。如果是覆盖对象，也是插入新的版本，而不是真正的覆盖。

* [sse](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#sseconfig)

  server-side-encryption，S3 的加密分为两类既客户端加密 和 服务端加密，前者用户自己管理秘钥。Min.IO 目前仅仅支持 SSE-S3，所有对象的秘钥一样，加密秘钥的主密钥会定期轮换重新加密对象的秘钥，使用AES256进行加密（GateWay支持更多类型，非本系列文章重点内容）。

* [tagging](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#taggingconfig)

  给 Bucket 设置 tag （可以是多个），标记Bucket 。

* [quota](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#quotaconfig)

  S3 基于容量的配额，分为两种类型，一种是 HardQuota 指硬配额，另一种 FIFOQuota 从bucket中删除旧文件的配额限制。

* [replication](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/bucket_metadata.md#replicationconfig--buckettargetconfig)

  Replicate 的作用是将写入的写对象复制到另一个存储桶，可以同区域复制（SRR）、跨区域复制（CRR）。

上述只是一些主要的子系统并不完全（例如 IAM），待后面遇到再详细介绍。



### 代码架构

![minio_code_arch](../../images/minio_code_arch.png)



##### 接口层

Min.IO Server 的代码架构如上图所示。最上面一层是 http 层，既接口，包括客户端与集群节点、集群内节点间的通信接口，分为6大类（角色权限管理、S3 接口、S3 各种子系统配置等）共计 171个接口（后面会重点介绍读写删除）。在接口层接收到请求之后，以 S3 的用户接口为例，都会有签名验证、权限管理验证、加解密、Quota（写类接口）、压缩等逻辑（会设计相应的配置子系统和子模块）。在真正存储数据之前，数据会先经过 [DiskCache](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/cache.md) 缓存。

##### 存储层

Min.IO 真正的存储层（ObjectLayer）在 DiskCache 之后，存储层类的继承关系如下：

![minio_objectlayer](../../images/minio_objectlayer.png)

ObjectLayer 是对外的接口（进程内跨模块调用），FSObjects是本地模式的存储层（非本系列文章介绍重点）。erasureZones 对应真个 Min.IO 集群，既若干个小的分布式集群，erasureSets 对应 erasureZones 中某一个小的分布式集群，Min.IO 集群扩容正是扩充的 erasureSets 。erasureObjects 对应 erasureSets 中某一个 纠删码组，一个 erasureSets 内包含若干个纠删码组（erasureZones、erasureSets、erasureObjects 对应关系与启动参数相关，后面介绍其计算逻辑）。erasureZones、erasureSets、erasureObjects 全部继承自 ObjectLayer 都会继承相应的接口（makeVol等），每层有其各自逻辑然后逐层调用，从而通过一套调用链完成相应的读写等操作。

xlStorage 对应一块磁盘，所有涉及文件等磁盘的操作都是由 xlStorage 完成。纠删码组的最小粒度是磁盘，既一个纠删码组是跨节点的多个磁盘组成。erasureSets 内部将磁盘（xlStorage）按纠删码分组，erasureObjects 当涉及到磁盘的操作时，通过函数指针从 erasureSets 获取 xlStorage 在进行操作，这种类关系涉及的一个好处是，erasureSets （一个 zone 内全部的纠删码分组）统一管理所有的磁盘，好处是避免将磁盘分配到 erasureObjects（一个纠删码分组）去管理带因一致性要求而带来，锁同步的复杂代码。

一个纠删码分组（erasureObjects）可能涉及多个跨节点的磁盘，xlStorage 分位本次磁盘（直接在本地操作），还有一种是跨节点的客户端，通过API接口实现远程交互。

erasureZones、erasureSets、erasureObjects、xlStorage 的对应抽象关系如下：

![minio_erasure](../../images/minio_erasure.png)



##### 子系统

在整体流程中除了接口层和存储层，还会涉及到数据修复、分布式锁（读写一致性考虑），以及为了兼容 S3 语义的 对象锁、生命周期等，在 Min.IO 中都有一个对应的子系统或者子模块去完成相关逻辑。除了之前介绍的 S3 需要的子系统，Min.IO 还会涉及以下重要的子模块：

* [DiskCache](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/cache.md)

  Cache 是在 Zone 上面的一层缓存，在http handler 中调用 Cache 的 API 接口，Cache 失败之后才会走 Zone 的接口逻辑。Cache 与 普通数据盘相比如果介质上有较好的提升，性能会有较多的提升，如果介质一样，与没有cache相比节省了 RS 编码和网络请求时间。

* [DistLock](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/dist_lock.md)

  Min.IO 中所有的存储节点都是对等的架构，不区分主从，min.io 使用分布式锁的方式保证数据的同步，例如：PutObject、GetObject 过程中都使用了分布式锁保证同一时刻只有一个请求可以处理同一个 Object。分布式锁除了在数据的读写之外，在内部一些协调逻辑中也有使用到。

* [Heal](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/heal.md)

  Min.IO 数据（Object）级别的数据修复，由于使用的 RS 编码的方式，所以没有副本修复（Set 间采用 Hash 分配 Objet，也没有容量负载均衡的逻辑）。

* [DataCrawler](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/minio/subsys/data_crawl.md)

  DataCrawler 作用是收集所有 bucket 的容量使用情况，然后将结果（dataUsageCache）作为对象本身写入到集群中。作为容量查询和 FIFOQuota 使用，同时会进行真正的数据删除。



### 参数解析



### 启动流程



### 主要流程介绍

##### GetObject

##### PutObject

##### DeleteObject

