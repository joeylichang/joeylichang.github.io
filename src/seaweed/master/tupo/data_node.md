# DataNode

```go
type DataNode struct {
	NodeImpl
	volumes      map[needle.VolumeId]storage.VolumeInfo			// vid -> VolumeInfo
	Ip           string
	Port         int
	PublicUrl    string
	LastSeen     int64 // unix time in seconds
	ecShards     map[needle.VolumeId]*erasure_coding.EcVolumeInfo
	ecShardsLock sync.RWMutex
}

func NewDataNode(id string) *DataNode {
	s := &DataNode{}
	s.id = NodeId(id)
	s.nodeType = "DataNode"
	s.volumes = make(map[needle.VolumeId]storage.VolumeInfo)
	s.ecShards = make(map[needle.VolumeId]*erasure_coding.EcVolumeInfo)
	s.NodeImpl.value = s
	return s
}
```



###### VolumeInfo

```go
type VolumeInfo struct {
	Id               needle.VolumeId
	Size             uint64
	ReplicaPlacement *ReplicaPlacement
	Ttl              *needle.TTL
	Collection       string
	Version          needle.Version
	FileCount        int
	DeleteCount      int
	DeletedByteCount uint64
	ReadOnly         bool
	CompactRevision  uint32
	ModifiedAtSecond int64
}
```

Master 内部也会维护维护一个volume的信息，该结构体在volume_server也会维护。

