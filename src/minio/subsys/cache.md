### Cache

##### 介绍

Cache 是在 Zone 上面的一层缓存，在http handler 中调用 Cache 的 API 接口，Cache 失败之后才会走 Zone 的接口逻辑。下面有几点需要说明：

1. 跨节点缓存：min.io 架构中所有节点都是对等的，请求可以打到任意节点，而且缓存都是本地缓存没有跨节点交互，所以一份数据可以被缓存在多个节点上（只要节点满足同一份数据被访问一定次数就会被在本地缓存），甚至缓存的节点上面没有相关对象的任何数据，会存在多分缓存数据不一致的问题，cache 的失效时间（用户自定义）在这里起到了至关重要的作用。
2. 接口：Cache 支持六个接口，PutObject（写元数据和数据）、GetObjectInfo（读取元数据）、GetObjectNInfo（读取数据和元数据）、CopyObject、DeleteObject、DeleteObjects。重点是前三个接口，PutObject、GetObjectNInfo 读取了真正的数据进行 cache hit or miss 的更新。
3. 分布式锁：PutObject、GetObjectNInfo 读取数据之前同样会获取全局的分布式锁才能进行操作，保证了操作的原子性，这部分与 zone 部分的逻辑保持一致，不至于操作期间数据有变化，与zone层的数据操作语义保持一致。
4. 介质：Cache 与 普通数据盘相比如果介质上有较好的提升，性能会有较多的提升，如果介质一样，与没有cache相比节省了 RS 编码和网络请求时间。



##### Cache 数据结构

```go
type cacheObjects struct {
	cache []*diskCache 	// cache 的磁盘，或者说路径
	exclude []string 	// 不被cache 的路径
	after int 			// 访问到一定次数之后才进行cache
	migrating bool 		// 元数据是否需要升级的标志位
	migMutex sync.Mutex

	cacheStats *CacheStats
		type CacheStats struct {
			BytesServed  uint64 	// cache 的大小
			Hits         uint64 	// 命中次数
			Misses       uint64 	// 未命中技术
				GetDiskStats func() []CacheDiskStats 	// 针对磁盘的统计
						type CacheDiskStats struct {
							UsageState int32 		// high value is '1', low is '0'
							UsagePercent uint64
							Dir          string
						}
						// high >= quotaPct * highWatermark ? true : false
						// low  <= quotaPct * lowWatermark ? true : false
				}
}

type diskCache struct {
	online       uint32 	// 1 代表在线，0 代表下线
	purgeRunning int32 		// 是否在 gc

	triggerGC     chan struct{} 	// 触发 gc 的通信管道
	dir           string         	// caching directory

  stats         CacheDiskStats 	// disk cache stats for prometheus
      type CacheDiskStats struct {
        UsageState int32 		// high value is '1', low its '0'
        UsagePercent uint64
        Dir          string
    }
		// high >= quotaPct * highWatermark ? true : false
		// low  <= quotaPct * lowWatermark ? true : false

	quotaPct      int         // 缓存容量 = quotaPct * 磁盘容量
	pool          sync.Pool 	// 4K 的内存空间对象，与直接IO大小对应
	after         int 				// 应该是被访问了 after 次之后，才进行cache
	lowWatermark  int 				// gc 的停止点 quotaPct * lowWatermark
	highWatermark int 				// 最大空间 quotaPct * highWatermark
	enableRange   bool 				// 是否支持 分片cache
	nsMutex *nsLockMap 			// 分布式锁，下面是回调
	NewNSLockFn func(ctx context.Context, cachePath string) RWLocker
}
```

cacheObjects 是当前节点的全部 Cache 信息的集合，也是 Cache 的入口。diskCache 针对的是单块磁盘的 cache 抽象。上述信息都是根据 cache.config 在初始化阶段完成的。

类比，数据目录，cache 目录也有自己的元数据，既 format.json，在 /cachepath/.minio.sys 目录下，内部结构如下：

```go
type formatCacheV2 = formatCacheV1
type formatCacheV1 struct {
	formatMetaV1
	type formatMetaV1 struct {
		Version string 	// Version of the format config
		Format string 	// backend format type, we force to xl
		ID string 		// deployment id 
	}
	Cache struct {
		Version string 	// Version of 'cache' format.
		This    string 	// disk uuid.
		Disks []string 	// input disk order generated the first time when fresh disks were supplied.
		DistributionAlgo string 	// hashing algorithm
	} 		// Cache field holds cache format.
}
```



##### 主要流程

###### PutObject

Put Object 和 ObjectInfo

1. 获取cache disk。

   1. index = crchash(bucket/object) / len(diskcaches)。
   2. 从 index 开始之后开始找，直到找到 object 在那个 disk ，直接返回。
   3. 没有找到，则是从index 开始第一个 online 的disk。

2. formate 升级ing、cache 磁盘容量不够、需要 SSE、S3语义层带有锁（用户态对象锁、对象持有）、cache exclude目录，上述情况都不会写入cache，直接写入走 zone 的接口进行写。

3. 经过上述约束检查之后，同步写入数据磁盘，然后异步写入 cache。

   1. 如果超过 highwarter ，则通知 gc 协程进行 gc。

   2. 获取分布式锁，才能进行写（写缓存仍然需要全局同步）。

   3. 如果没有该对象的 cache，则保存 meta 直接返回。

      1. /cachedisk/sha256hash(object/object)/cache.json

      2. ```go
         type cacheMeta struct {
         	Version string   	// "1.0.0"
         	Stat    StatInfo 	// 包含 数据大小size 和 modtime
         		
           // 包含：校验和算法名，直接io 的数据块大小 4KB
         	Checksum CacheChecksumInfoV1 	
           // 主要是 etag ，全部来自 userdefine
         	Meta map[string]string 		
           // 针对分片 cache，<offset-length, uuid(文件名)>
         	Ranges map[string]string 	
           // hit 计数
         	Hits int  					 
         }
         ```

   4. 如果有 cache_meta，但是还没有 cache，重新 save_meta 内部会 hit++。

      1. 通过 etag 判断是否是同一个对象，用户自定义的唯一标识。
      2. hit < after 则直接返回。

   5. 如果是多分片数据（应该是 mutilpart），则进行 putRange：

      1. putRange 之后直接返回，步骤6 之后续逻辑是 cache 单独的 objectinfo。
      2. 整体流程大致与单 object 的逻辑一致（bitrot 写入到一个指定的文件中）。
      3. 文件 /cachedir/sha256hash(object/object)/uuid。
      4. meta 中 range 记录 <offset-length, uuid> 的映射。

   6. 判断写入数据之后磁盘容量是否超过 pct * high_water。

   7. 对数据进行加密部分的逻辑（KMS 加密，min.io server 不涉及）。

   8. bitrot 写入 /cachedisk/sha256hash(object/object)/part.1。

   9. 重新 save_meta ，更新 hit。

   **注意：cache_meta 是先有的，达到一定标准之后才 cache 。**



###### GetObjectInfo

读取 ObjectInfo

1. 是否在 cache 的 exclude 里面。

2. 如 PutObject 逻辑获取 cache 磁盘位置，通过 diskcache 的 state 接口获取 cachedObjInfo。

3. 根据 cachedObjInfo 判断是否过期（UserDefined 中 cache-control 内部设置的 cache 相关控制信息），没有则返回 cachedObjInfo。

4. 到此，cache 没有命中，调用 zone 的接口。

   **注意：**

   1. 结束之后会更新 cache 的统计信息。
   2. Get 并没有导致数据从数据盘写入到 Cache 中，也没有更新 hit 等统计信息。

   

###### GetObjectNInfo

读取 Object 和 ObjectInfo

1. 是否在 cache 的 exclude 里面。
2. 如 PutObject 逻辑获取 cache 磁盘位置。
3. 从 cache 中读取数据：
   1. 分布式锁。
   2. 根据是否是存在 part.1 文件判断是否是分片数据，组织成 objInfo。
   3. 如果是分片数据，则根据 size 计算出是哪个 rang 的文件，组织成 rngInfo。
   4. 针对目录对象进行处理，没有读数据部分，直接返回。
   5. 新的协程异步 bitrot 读。
   6. 更新统计信息hit 或者 miss等。
4. 从 cache 中读取数据失败，则走 zone 接口读取数据，之后写入cache：
   1. 用户没有设置不 cache 标记、没有S3语义层带有锁（用户态对象锁、对象持有）、容量够用、hit > after。
   2. 异步写入 cache。



###### CopyObject

1. 是否在 cache 的 exclude 目录。

2. 不是原地copy（原地copy会更新元数据），直接调用 zone 的 copy 接口

3. 如果是原地 copy，根据 user_define 的 cache 策略决定是否过期，没有cache策略或者过期则删除cache。

4. 进行原地 copy。

   **注意：如果 cache 策略仍然生效，则 copy 之后的cache 仍然存在，但是数据已经copy到目标bucket或者 object 了**

   

###### DeleteObject

先删除磁盘再删除 cache



###### DeleteObjects

批量删除，循环调用上面 DeleteObject 逻辑



##### GC

1. 30min 遍历所有的磁盘进行gc（另一种触发单盘 gc 的场景是，GetObjectNInfo 中 put 数据的时候超过 pct * high_water）。

2. 计算需要删除的数据量（删除多少能够达到 pct * lowwater）。

3. 更新 gc 原子变量，defer 释放。

4. 开始遍历删除 cache 文件。

   1. 用户自定义了 cache 策略：

      1. onlyIfCached = true 		表示永不过期
      2. noStore = true 				表示不用cache
      3. revalidated = true 			表示需要激活 cache 才能生效
      4. sMaxAge、maxAge、minFresh 	表示有效时间，最后一个 mod 超过这个时间 则过期
      5. expiry 						表示过期时间，maxStale + now 超过这个值，则过期（注意：跟 mod 时间没关系）

   2. 目录已经 90 天没有更新，则直接删除。

   3. 用户没有定义 cache 策略，且没有超过90天，对文进行打分：

      1. (now - mod) * (1 + 0.25*szWeight + 0.25*hitsWeight)
      2. szWeight = size * 1/toFree (0-1之间)
      3. hitsWeight = 1 - hits / maxHits（100 固定值） (0-1之间)

      **注意：**

      1. 主要因素是最后一次访问时间距离现在有多久，最少占比 2/3。
      2. 其次因素是文件大小、命中次数占比一样，最多 1/6。

   **注意**

   1. 上述步骤4中三种情况，遍历每个文件都会做上述判断，满足前两条就会删除，删除之后判断剩余容量是否满足要求，满足直接返回，否则继续指导劝募遍历完，对打分进行排序，从分高的文件开始删除，直到删除安全容量以下。