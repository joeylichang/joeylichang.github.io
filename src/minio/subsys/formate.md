### Format.json

```go
type formatErasureV3 struct {
	formatMetaV1
	Erasure struct {
		Version string `json:"version"` // 最新的版本是 3 
		This    string `json:"this"`    // disk uuid，初始化时是一个随机值
		Sets [][]string `json:"sets"` 	// 一维 = 分几组EC，二维 = 每组EC的分块数量
    																// 内容是磁盘的随机 ID
		DistributionAlgo string `json:"distributionAlgo"`
    																// object 在zone内，判断所述 Set 时的 hash 算法名
	} `json:"xl"`
}

type formatMetaV1 struct {
	// Version of the format config.
	Version string `json:"version"` 	// 默认值是 1，formate config 的版本号
	Format string `json:"format"`			// 存储类型，单机模式（fs），分布式模式（xl），这里关注后者
	ID string `json:"id"` 						// 部署 ID， deploymentID
}

/*
 * 注意：
 * 	1. formatErasureV3.Erasure.This = 对应 formatErasureV3.Erasure.Sets 相应位置的 UUID
 * 	2. 如果指定的了 deploymentID（环境变量） ，则赋值给 formatErasureV3.formatMetaV1.ID，否则随机值
 * 	3. formatErasureV3.formatMetaV1.ID 会赋值给 goableDeploymentId
 */
```



上述结构体是内存中的结构，会持久化在磁盘上，每一个数据盘都会有自己的 format.json，在目录：/diskpath/.minio.sys/format.json 。

在初始化 ObjectLayer 对象时，会初始化 ErasureZones ，在其内部会根据传入的参数（参数经过前面介绍的处理和预计算之后）初始化 format.json，内部成员含义结合前面 ObjectLayer、ErasureZones、ErasureSets、ErasureObjects、Storage 部分的介绍便于理解，下面几点值得注意：

1. 在 ErasureZones 初始化阶段会根据处理之后的参数（EndpointZones，Host:DiskPath 经过 EC分组之后的组织形式）遍历所有的 zone，初始化 format.json。
2. 一个 format.json 内部记录的是一个 zone 内所有的 set 信息。
3. **只有所有 zone 中的第一个节点进行 formate.json 的初始化工作（内部有同步机制，使得其他节点等待 format.json初始化完毕）。**
4. 所以一个 format.json 只记录一个 zone 的拓扑信息或者元数据，既每个zone 内的元数据是隔离的（故，在多个zone场景中put时，会先遍历所有的 zone 查看之前在那个zone在写入，没有的话才会根据容量再选一个）。

