# VolumeGrowOption

```go
type VolumeGrowOption struct {
	Collection         string
	ReplicaPlacement   *storage.ReplicaPlacement
	Ttl                *needle.TTL
	Prealloacte        int64
	DataCenter         string
	Rack               string
	DataNode           string
	MemoryMapMaxSizeMb uint32
}

func (vg *VolumeGrowth) findVolumeCount(copyCount int) (count int) {
	switch copyCount {
	case 1:
		count = 7
	case 2:
		count = 6
	case 3:
		count = 3
	default:
		count = 1
	}
	return
}

// 根据replicate策略，对创建replication有一个预申请的buffer，放大策略如findVolumeCount
func (vg *VolumeGrowth) AutomaticGrowByType(option *VolumeGrowOption, grpcDialOption grpc.DialOption, topo *Topology) (count int, err error) 

// 调用targetCount 次 findAndGrow，
func (vg *VolumeGrowth) GrowByCountAndType(grpcDialOption grpc.DialOption, targetCount int, option *VolumeGrowOption, topo *Topology) (counter int, err error)

// 调用findEmptySlotsForOneVolume 选取合适的node，调用grow 通知volume_server创建volume
func (vg *VolumeGrowth) findAndGrow(grpcDialOption grpc.DialOption, topo *Topology, option *VolumeGrowOption) (int, error)

// 根据replicate策略 逐层（dc、rack、dn）选取合适的node
func (vg *VolumeGrowth) findEmptySlotsForOneVolume(topo *Topology, option *VolumeGrowOption) (servers []*DataNode, err error)

// 通知servers创建volume
func (vg *VolumeGrowth) grow(grpcDialOption grpc.DialOption, topo *Topology, vid needle.VolumeId, option *VolumeGrowOption, servers ...*DataNode)

```

调用assign接口事会调用AutomaticGrowByType（后续介绍）。调用链关系：AutomaticGrowByType -> GrowByCountAndType -> findAndGrow -> findEmptySlotsForOneVolume / grow。