# Topology

Topology 是master维护的拓扑信息的全局视图。

```go
type Topology struct {
	vacuumLockCounter int64				// 当volume删除数据超过闸值时，要进行压缩，压缩的并行度控制
	NodeImpl

        collectionMap  *util.ConcurrentReadMap	        // 协程安全的map，string->Collection(后续介绍)
	ecShardMap     map[needle.VolumeId]*EcShardLocations
	ecShardMapLock sync.RWMutex

	pulse int64					// volume_server 心跳上报的周期
	volumeSizeLimit uint64				// volume的大小限制，默认30
	Sequence sequence.Sequencer 	                // volumeID，从1开始递增
	chanFullVolumes chan storage.VolumeInfo		// 管道用于check volume是否写满，表示为不可写
	Configuration *Configuration	                // 拓扑信息的配置

	RaftServer raft.Server				// master节点rart的抽象
}



func NewTopology(id string, seq sequence.Sequencer, volumeSizeLimit uint64, pulse int) *Topology {
	t := &Topology{}
	t.id = NodeId(id)				// id = topu
	t.nodeType = "Topology"				// nodeImpl 指定节点类型
	t.NodeImpl.value = t
	t.children = make(map[NodeId]Node)		// 子节点一定是datacenter
	t.collectionMap = util.NewConcurrentReadMap()
	t.ecShardMap = make(map[needle.VolumeId]*EcShardLocations)
	t.pulse = int64(pulse)
	t.volumeSizeLimit = volumeSizeLimit

	t.Sequence = seq

	t.chanFullVolumes = make(chan storage.VolumeInfo)

	t.Configuration = &Configuration{}

	return t
}

```

