### DataCrawler

##### 介绍

DataCrawler 作用是收集所有 bucket 的容量使用情况，然后将结果（dataUsageCache）作为对象本身写入到集群中。DataCrawler 收集的容量信息有如下几个作用：

1. Quota 使用，特别是 FIFOQuota 类型每次收集完数据如果是 bucket 设置的 Quota 类型是 FIFOQuota 会进行数据删除（后面介绍）。
2. 支持查询容量使用的接口使用。
3. DataCrawler 在收集信息时，需要全量扫描数据，期间会对过期数据和副本（见 Replication 子系统部分）失败的数据进行删除和恢复。

DataCrawler 除了与 Quota 部分逻辑有交集，还与 dataUpdateTracker 子模块有交集。dataUpdateTracker 简单说（后面介绍）内部维护一个 boolmfilter 记录更新的 bucket/object。DataCrawler 在更新数据容量时会更新有变化或者有 LifeCycle 设置的数据（可能过期），boolmfilter 正式记录 bucket/object 是否有变化的，但是 boolmfilter 一直不更新效果会严重下降，所以 min.io 内部有一套完整的 boolmfilter 更新机制（后续介绍）。

DataCrawler 需要保证集群中是单点运行的，否则要么有大量协调工作（将任务分给不同节点执行），要么有大量重复（每个节点统计的数据有重复，由于需要扫描数据，对 CPU 等资源消耗还是比较大的）。所以 DataCrawler 有集群主节点（leader，在程序启动阶段通过分布式锁选主）进行执行，每 5 min 执行一次。



##### 主要数据结构

```go
type dataUsageCache struct {
	Info  dataUsageCacheInfo 	
  	type dataUsageCacheInfo struct {
      // Name of the bucket. Also root element.
      Name        string
      LastUpdate  time.Time
      NextCycle   uint32
      BloomFilter []byte               
      lifeCycle   *lifecycle.Lifecycle 
    }
  
  Cache map[string]dataUsageEntry 	// key 是hash(路径名)
  	// 代表了当前节点及其子节点的总统计信息
  	type dataUsageEntry struct {
      // These fields do no include any children.
      Size     int64
      Objects  uint64
      ObjSizes sizeHistogram 	// object 大小的柱状图统计数据
				type sizeHistogram [dataUsageBucketLen]uint64 // dataUsageBucketLen == 7
      	var ObjectsHistogramIntervals = []objectHistogramInterval{
          {"LESS_THAN_1024_B", 0, humanize.KiByte - 1},
          {"BETWEEN_1024_B_AND_1_MB", humanize.KiByte, humanize.MiByte - 1},
          {"BETWEEN_1_MB_AND_10_MB", humanize.MiByte, humanize.MiByte*10 - 1},
          {"BETWEEN_10_MB_AND_64_MB", humanize.MiByte * 10, humanize.MiByte*64 - 1},
          {"BETWEEN_64_MB_AND_128_MB", humanize.MiByte * 64, humanize.MiByte*128 - 1},
          {"BETWEEN_128_MB_AND_512_MB", humanize.MiByte * 128, humanize.MiByte*512 - 1},
          {"GREATER_THAN_512_MB", humanize.MiByte * 512, math.MaxInt64},
        }
      
      /*
       * 1. key 同样是hash(路径名)，value 是空结构体
       * 2. 主要是记录有哪些子目录，struct 可能是为了后期扩展
       * 3. 根据子目录的 key hash(路径)能查询到 dataUsageEntry 的详细信息
       */
      Children dataUsageHashMap 	
      	type dataUsageHashMap map[string]struct{}
    }
}
```

上面介绍的 dataUsageCache 正是 DataCrawler 收集的数据并且通过正常的用户端接口（EC 编码）写入到 /diskpath/.minio.sys/buckets/.usage.json 。

在正是介绍 DataCrawler 之前需要明确以下即可对象存储的数据和作用（全部使用用户接口，EC编码写入集群）：

1. /diskpath/.minio.sys/buckets/.bloomcycle.bin 
   1. 记录了下一个 boolmfilter 的序号（详见 boolmfliter 管理子系统介绍）。
2. /diskpath/.minio.sys/buckets/.usage.json
   1. dataUsageCache 数据
   2. 最终的结果
3. /diskpath/.minio.sys/buckets/.usage-cache.bin
   1. dataUsageCache 数据
   2. 与 .usage.json 相同，是当前 Set 的结果
4. /diskpath/.minio.sys/buckets/${Buckets}/.usage-cache.bin 
   1. dataUsageCache 数据
   2. 是每个 bucket 的结果



##### DataCrawler 主要逻辑

min.io 启动阶段完成所有的子系统初始化之后，会调用 startBackgroundOps 启动后台任务。startBackgroundOps 内部会通过分布式锁选出全局的 leader 节点，只有 leader 节点才会启动后台任务（其他节点会周期抢主）。startBackgroundOps 选出 leader 之后，由 leader 节点开始启动 GlobalHeal（详见 heal 子系统介绍）和 initDataCrawler 。下面重点看一下  initDataCrawler 的逻辑：

1. 启动一个新的协程运行 runDataCrawler。

2. runDataCrawler 内部先获取 nextBloomCycle （一个 boolmfilter 的序号，后面介绍 boolmfilter 管理模块）。

3. 周期性（5min）进行数据收集：

   1. 通过 Notification 子系统通知所有节点更新 boolmfilter （新的 boolmfilter 编号为 nextBloomCycle）。
   2. 调用 zone 的 CrawlAndGetDataUsage 接口收集数据，收集的数据通过管道传给另一个后台协程（storeDataUsageInBackend）写入 /diskpath/.minio.sys/buckets/.usage.json 对象内部。
      1. **注意：CrawlAndGetDataUsage 是一个漫长的过程，每次都是收集上来一部分新的数据覆盖旧的数据（dataUsageCache），然后整体去覆盖 .usage.json 的内容，既storeDataUsageInBackend 内部写入的数据在一个周期内总是一部分是新的一部分数上个周期的，是一个持续的动态过程。**
      2. storeDataUsageInBackend 协程内部等待管道的数据（dataUsageCache），进行序列化然后调用用户接口进行 EC 编码写入 .usage.json 对象。
   3. nextBloomCycle ++，写入 /diskpath/.minio.sys/buckets/.bloomcycle.bin 待下次重启使用。

   

收集数据的核心逻辑在 CrawlAndGetDataUsage，下面看一下详细过程。

1. 遍历所有的 erasureZones 以及 erasureSets，在每一个 erasureSets 中随机选一块磁盘，ListBucket 获取当前 zone、set 内的全部 Bucket。
2. 新启动一个协程（所有 zone 的 所有 set，并发执行）：调用 Set 内 erasureObjects 的 crawlAndGetDataUsage 接口（参数 Buckets）进行数据扫描。
3. 对于扫描的结果利用同步机制（锁）放到一个数据接口内（[]dataUsageCache）。
4. 新启动一个协程：对于收集的结果，定期进行合并，并且通过管道传回上一层 runDataCrawler 中写入 /diskpath/.minio.sys/buckets/.usage.json 对象。
5. 在所有的扫描任务结束之后，遍历所有的 bucket ，调用 enforceFIFOQuotaBucket 根据配额进行数据删除（FIFOQuota 部分会消息介绍）。



下面看一下 erasureObjects 内部如何以 Bucket 为粒度进行信息的收集（crawlAndGetDataUsage）：

1. 加载上次收集的旧数据（dataUsageCache）
   1. /diskpath/.minio.sys/buckets/.usage-cache.bin
2. 挑选出需要进行收集数据的 bucket
   1. 新增的 bucket。
   2. 非新增的 bucket，有生命周期设置且生效中 || bf 为空 || bf 中有这个 bucket 的记录。
   3. 上述两种 bucket 都传入管道（后续处理）。
   4. **注意：其他不变化的 bucket 的cache 数据直接从加载的就数据中，读出来赋值到新的 cache 中（最为新一轮收集数据的结果）**
3. 开启一个新的协程，对于收集的记过进行落盘
   1. 周期性（30s）写入一次对象 /diskpath/.minio.sys/buckets/.usage-cache.bin 中
4. 遍历当前 Set 的所有磁盘，每个磁盘启动一个新的协程
   1. 每个协程等待步骤2 中管道的数据
      1. **注意：管道中的数据只能被消费一次，所以此处让每块磁盘启动一个任务，是将任务分拆到每次磁盘上减少压力**
   2. 加载当前 bucket 的数据，/diskpath/.minio.sys/buckets/${Bucket}/.usage-cache.bin
   3. 调用磁盘（xlstorge）的 CrawlAndGetDataUsage 接口收集数据
   4. 保存当前 bucket 的数据，/diskpath/.minio.sys/buckets/${Bucket}/.usage-cache.bin
      1. 该层只直接调用操作系统的接口遍历所有的 bucekt/object 两层目录，收集数据容量信息
      2. 先遍历一遍找出新的目录，针对新目录更新 cache 的 Size、Objects、ObjSizes
      3. 针对就目录，查看是否有生命周期 或者 boolmfilter ，没有的话不做任何更新，否则如上更新数据
      4. 获取对象大小的方式：读取每个对象的 xl.json 元数据文件，获取其 FileInfoVersions （针对 S3 版本控制的数据接口），然后遍历所有的 FileInfoVersions，获取对象的大小（如果有删除标记，则忽略），最后针对有 Replication 配置的 bucket 进行 healReplication 。
   5. 对于收集的数据同时通过管道返回上层 erasureZones 的接口中



##### FIFOQuota

在 erasureZones 层遍历任务结束之后，会遍历所有的 bucket，针对每一个 bucket 调用 enforceFIFOQuotaBucket 进行数据根据配额 和 配置进行删除数据。enforceFIFOQuotaBucket 作用是对于配置了 FIFOQuota 类型的 quota ，删除对于容量配置的相应大小的数据。

```go
type fileScorer struct {
	saveBytes uint64 	// toFree
	now       int64 	// 
	maxHits   int 		// 1
	// 1/size for consistent score.
	sizeMult float64 	1 / float64(saveBytes)

	// queue is a linked list of files we want to delete.
	// The list is kept sorted according to score, highest at top, lowest at bottom.
	queue       list.List
  	 file := queuedFile{ // queue 内部的数据结构
		 		name:      objInfo.Name,
			 	versionID: objInfo.VersionID,
				size:      uint64(objInfo.Size),
		  }
	queuedBytes uint64 	// 当前 queue 内部的数据大小
	seenBytes   uint64 	// 已经遍历的对象大小
}
```

遍历 bucket 下的对象（除了上锁的对象），将数据加入到 fileScorer，fileScorer 对不的队列会对所有的文件进行排序，排序标准与 cache 相同（之前介绍的数据 cache 子系统）

file.score = (f.now - objInfo.ModTimeAccTime.Unix()) * (1 + 0.25*szWeight + 0.25*hitsWeight)

hitsWeight := (1.0 - math.Max(0, math.Min(1.0, float64(hits)/float64(f.maxHits))))

szWeight := math.Max(0, (math.Min(1, float64(file.size)*f.sizeMult)))

根据打分从高到低进行删除，直到删除到配额以下位置。



##### dataUpdateTracker && BloomFilter

正如前面介绍 DataCrawler 在更新每个 bucket 的统计数据时，会通过 boolmfilter 判断是否有更新，没有更新会使用上一轮的统计数据，减少没必要的扫描消耗 CPU 资源。数据写入时，bucket/object 会加入到BloomFilter，这样在 DataCrawler 的时候回根据这个判断是否有更新，BloomFilter 的管理子系统正是 dataUpdateTracker 模块。

```go
type dataUpdateTracker struct {
	mu         sync.Mutex
	input      chan string 		// 接收 bucket/object 的管道，然后加入 boolmfilter
	save       chan struct{} 	
	debug      bool
	saveExited chan struct{}
	dirty      bool

	Current dataUpdateFilter
  	type dataUpdateFilter struct {
      idx uint64
      bf  bloomFilter
    }
	History dataUpdateTrackerHistory 	// []dataUpdateFilter
	Saved   time.Time
}
```

History 会记录历史的 bloomFilter ，但是只会记录 16个（默认值），每次更新会指定 Current 和 Oldest 记录 History 的区间。

dataUpdateTracker 启动其中会启动两个协程，第一个协程接收 input 的结果，然后加入 Current.bf.AddString 中记录该对象被更新过。另一个协程，每隔 5min 或者 接收到 Notification 的通知，然后将  dataUpdateTracker 序列化节后落盘，本地所有磁盘的 /diskpath/.minio.sys/buckets/.tracker.bin 都会落。