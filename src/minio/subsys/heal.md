### 数据修复

##### 介绍

min.io 数据恢复是数据（Object）级别的数据修复，由于使用的 EC 编码的方式，所以没有副本修复（Set 间采用 Hash 分配 Objet，也没有容量负载均衡的逻辑）。

###### 分层架构

数据修复的整体框架分为三部分，中间层是数据修复的调度以及中间状态信息的维护，主要的数据结构是 healSequence ，称作数据修复计划，修复计划与上层修读任务的来源有关分为 admin 接口（修复内容来自接口参数）、程序自身的周期全量扫描任务（扫描出有损坏的数据），每种任务来源对应各自的修复计划。修复计划根据任务来源描述的信息进行扫描相应的数据（admin 接口指定 或者 全量扫描），然后生成需要修复的 healtask 子任务发送给最后一层的任务队列，后台协程从队列中取出任务进行修复。



###### 任务分类

无论是来自 admin接口 还是周期任务扫描全量数据生成的任务，所有的任务分位三类，既 healDiskFormat、HealBucket、HealObject。

任何节点在启动阶段都会启动中间层的数据修复框架 和 最后一层从队列取出任务进行修复的执行逻辑。但是全量扫描任务只有集群中的 leader（通过抢占资源锁） 才能执行。

其中 healDiskFormat 是修复 /diskpath/.minio.sys/format.json 文件，每个节点只检查当前节点的磁盘的 formate.json 文件，然后进行修复。不需要非得主节点才能进行修复。

**注意：**

1. 全量数据修复应该是考虑避免重复修复，必须保证单点执行。
2. formate.json 不是ec 编码，且是集群拓扑信息（EC 编码、Zone、Set等）重要的元数据，有必要进行修复。



###### 数据修复模式

修复任务的来源除了上述来源以外，GetObject 期间发现数据解码发生错误，也会将其封装成任务进行修复。

此时，指定的修复模式是 HealDeepScan，该模式下会使用 bitrot 对数据内容级别检测。之前介绍的都是 HealNormalScan，单纯的检查文件是否存在，不对内容进行校验。





##### 核心数据结构

```go
type healSequence struct {
  /*
   * 两种任务的来源
   * 1. bucket, object 来自 admin 接口的任务必须制定 
   * 2. sourceCh 后台定期（30 days）的扫描全量数据生成的任务来源
   */
	bucket, object string
	sourceCh chan healSource
	respCh chan healResult		// 每个任务执行完毕的结果，通过管道传回主控协程
  
  // 是否将heal task 执行结果加入 currentStatus 中，仅针对 globalAllHealState 起作用，admin 接口会查询这个信息
	reportProgress bool 			
	startTime time.Time
	endTime time.Time
  
  // 客户端信息，globalBackgroundHealState是固定值，globalAllHealState中有可能 client 需要查询结果，作为身份验证
	clientToken, clientAddress string
  // 是否强制执行，如果存在path的任务会 stop，重新开始，仅针对 globalAllHealState 起作用
	forceStarted bool				

  // globalBackgroundHealState 是固定值， globalAllHealState 可以用户指定参数
	settings madmin.HealOpts 		// 用自定义的一些参数
  	type HealOpts struct {
      Recursive bool        	// 是否递归，globalBackgroundHealState 为true
      DryRun    bool         
      Remove    bool         	// 不满足 read-quorum 要求的数据是否直接删除（删除未完全的数据）
      ScanMode  HealScanMode 	// 扫描模式（后面介绍）， globalBackgroundHealState 为 madmin.HealNormalScan
    }
  
	currentStatus healSequenceStatus		// 记录了修复计划的进度，和每个子任务的执行结果（Items）
  	type healSequenceStatus struct {
      // summary and detail for failures
      /*
       * Summary:
       * 	healNotStartedStatus  = "not started"
       *	healRunningStatus     = "running"
       *	healStoppedStatus     = "stopped"
       *	healFinishedStatus
       */
      Summary       healStatusSummary 
      FailureDetail string            
      StartTime     time.Time         

      HealSettings madmin.HealOpts  // settings for the heal sequence
      Items []madmin.HealResultItem // slice of available heal result records
    }
  
	// 修复的结果，返回给主控协程的通道，仅用于 globalAllHealState
	traverseAndHealDoneCh chan error 
	// canceler to cancel heal sequence.
	cancelCtx context.CancelFunc
	// the last result index sent to client
	lastSentResultIndex int64

	// Number of total items scanned against item type
	/*
	 * 每种类型scan的计数，无论成功失败
	 * 类型分为：bucket、object、meta
	 * 仅在 sourceCh 模式下进行统计
	 */
	scannedItemsMap map[madmin.HealItemType]int64
	// Number of total items healed against item type
	healedItemsMap map[madmin.HealItemType]int64
	// Number of total items where healing failed against endpoint and drive state
	healFailedItemsMap map[string]int64
  
	/*
	 * 每完成一个 heal 任务之后更新一下时间
	 *  仅在 sourceCh 模式下更新
	 */
	lastHealActivity time.Time
	// Holds the request-info for logging
	ctx context.Context
	// used to lock this structure as it is concurrently accessed
	mutex sync.RWMutex
}
```

healSequence 代表一个扫描计划，一个扫描计划在施行的过程中会生成多个 healtask 给后台协程去进行修复（后面介绍）。healSequence 的来源有两种，每种对应 healSequence 内部不同的成员变量（既，在每种来源中有的成员变量是不起作用的，详情见上面注释），一种是 background（定时任务扫描全部数据），另一种是来自 admin 接口。前者的生成的任务通过 sourceCh，将任务信息传进来经过流程处理之后加入后台协程的任务队中。后者在初始化阶段需要指定 bucket, object ，在指定前缀的目录上进行扫描成成任务加入后台修复协程的队列中。



两种来源的扫描计划，分别由两个全局 map 进行管理，既 globalAllHealState、globalBackgroundHealState，全部是 allHealState 类型。

```go
type allHealState struct {
	sync.Mutex

	// map of heal path to heal sequence
	healSeqMap map[string]*healSequence
}
```

allHealState 在初始化阶段会启动一个后台任务，周期性（5min）检查所有的 healSequence 是否结束，并且结束了是否在 10min 以上，都满足的话会对其进行删除。除此之外 allHealState 还支持全局扫描计划的查询和控制接口。



无论是 globalAllHealState 还是 globalBackgroundHealState 的扫描计划，在扫描过程中遇到需要修复的数据都会生成对应的 healTask 通过管道传递给后台协程的任务队列中，然后从队列中取出任务进行数据修复。

```go
type healTask struct {
	bucket    string						// 扫描需要或者 admin 接口传入需要修复的 bucket
	object    string						// 扫描需要或者 admin 接口传入需要修复的 object
	versionID string						// 版本号，详情见 S3 version 部分介绍
	opts      madmin.HealOpts 	// 用户设置的参数，来自上述 healSequence.settings
	responseCh chan healResult 	// 修复结果返回的管道，返回主线程
}
```



##### 周期扫描全量任务进行数据修复

在程序初始化阶段会先初始化  globalAllHealState 、 globalBackgroundHealState ，然后初始化后台协程等待任务的达到在进行相应的数据修复，最后会针对 globalBackgroundHealState 初始化一个后台的扫描计划，既 newBgHealSequence，其重要作用是主要是针对  healSequence 中 globalBackgroundHealState 关心的成员变量进行初始化，这样后台全量扫描任务周期执行时会会启动这个计划。

至于 globalAllHealState 就无需提前生成扫描计划，因为是根据 admin 接口生成的。两者的核心逻辑相同，在这里重点介绍一下周期扫描全量数据进行修复的流程。

###### newBgHealSequence

首先看一下，周期全量扫描全量数据的扫描计划的定制化 healSequence

```go
hs := madmin.HealOpts{
		Remove:   true,
		ScanMode: madmin.HealNormalScan,
	}

&healSequence{
		sourceCh:    make(chan healSource),
		respCh:      make(chan healResult),
		startTime:   UTCNow(),
		clientToken: bgHealingUUID, 		// 特殊的 client token
		settings:    hs,
		currentStatus: healSequenceStatus{
			Summary:      healNotStartedStatus,
			HealSettings: hs,
		},
		cancelCtx:          cancelCtx,
		ctx:                ctx,
		reportProgress:     false,
		scannedItemsMap:    make(map[madmin.HealItemType]int64),
		healedItemsMap:     make(map[madmin.HealItemType]int64),
		healFailedItemsMap: make(map[string]int64),
	}
```

上述涉及的成员变量是 globalBackgroundHealState 所需要的，既没有初始化的变量都是 globalAllHealState 独享的。



###### 周期扫描全量数据

在程序初始化阶段会调用 newBgHealSequence，生成后台扫描数据的计划，然后调用 globalBackgroundHealState.LaunchNewHealSequence 将 globalBackgroundHealState 加入的 map 中，其中 map 的key 是空字符串。

LaunchNewHealSequence 内部会调用 globalBackgroundHealState.healSequenceStart（内部调用healItemsFromSourceCh）等待来自 healSequence.sourceCh 的任务到来，接收到任务之后调用 globalBackgroundHealState.queueHealTask，根据扫描信息初始化 healTask ，然后将 healTask 加入到后台的任务中等待执行（后面介绍）。

下面来看一下扫描全量数据的流程：

1. 程序启动阶段会抢占全局锁（资源：/diskpath/.minio.sys/leader），抢占失败的节点会定期 1小时重试，成功的为全局主节点，只有主节点才负责数据的修复。
2. 无限循环等待 globalBackgroundHealState 中 BgHealSequence 初始化完毕（通过 token 区分）。
3. 30天，遍历所有的 zone 及其 Set，调用 healErasureSet 进行全量数据扫描。
4. healErasureSet 主要流程：
   1. buckets = ListBuckets + /diskpath/.minio.sys/config + /diskpath/.minio.sys/buckets
   2. 遍历全部的 buckets。
   3. 调用 xlstorage 的 WalkVersions 接口，将想用目录下面所有的文件（递归）组织成 FileInfoVersion 接口。
   4. 拿到所有 EC 分块数量的目录子项信息（FileInfoVersion）进行比对（是否有文件缺失等）。
   5. 如果获取的文件 EC 编码数量少于全量的 EC 分块数量，则组织成healSource（见前面介绍），传入 healSequence.sourceCh 待后台进行修复。

**注意：newBgHealSequence 和 初始化后台数据修复任务，不仅仅是主节点（只有全量数据扫描是主节点），其他节点也会初始化。**



除了上述情况（buckets + .minio.sys/config + .minio.sys/buckets）之外，还有一种 disk 上面 format.json 文件有问题时，也需要修复，并且除了修复 format.json 文件，还需要修复其上的数据。

在程序初始化阶段会启动一个后台任务（monitorLocalDisksAndHeal）周期性（3min）的扫描 format.json 文件情况。monitorLocalDisksAndHeal 主要流程：

1. 3min 周期性，遍历全部的zone，关注本地的磁盘，尝试加载 /diskpath/.minio.sys/format.sjon，对于加载失败的节点记录下来。
2. 组织成 healSource （bucket 为 "/" 特殊标记），发送给 sourceCh 进行修复。
3. 然后记录出问题的disk，定位器zone 和 set，然后调用上述 healErasureSet 进行修复。



##### 后台修复数据

后台接收到修复任务进行修复，分位 healDiskFormat、HealBucket、HealObject三种类型。

###### healDiskFormat

1. 加载目前全部可读的 format.json。
2. 根据启动参数重新计算新的 format.json，基于上述旧的信息。
3. 生成新的全部磁盘的 format.json ，写入磁盘。
4. 最后，通过 notify 子系统重新加载先关磁盘的  format.json 文件。



###### HealBucket

直接调用 zone 相应的接口，经过 Set 透传调用 Obejct 的相关接口（逐层遍历）， Obejct 的相关接口内部调用 StatVol（xlstorage的接口） 请求，只针对 errVolumeNotFound 错误进行处理，重新 makevol。

既，HealBucket 只是修复 bucket 目录，内部数据并没有处理。



###### HealObject

同样调用 zone 相应接口（内部遍历所有的zone），调用 Set 相应接口（hash计算），然后调用 Object 的 Heal接口。

1. 如果 object 参数有 "/" 说明是路径，则恢复路径，既 mkdir bucket/object。
2. 读取 bucket/object/xl.meta 文件内容，用其判断是否是悬空对象（多余一半的分片有问题，无法恢复），进行删除（根据参数决定，remove 是否为 true）。
3. 获取全部磁盘上 bucket/object/xl.meta 上的最后修改时间
   1. 如果最新的修改时间少于一般，则故障，此时返回 err
   2. 如果返回的结果全部是错误，则删除对象（根据传入参数，remove 是否为 true）
4. 到此，少于一半磁盘获取最后修改时间有问题，此时进行 对象修复（EC 编码修复）。