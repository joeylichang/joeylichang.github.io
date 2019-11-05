# Store

```go
type Store struct {
	MasterAddress       string					// master addr
	grpcDialOption      grpc.DialOption
	volumeSizeLimit     uint64 					// read from the master, 默认 30G
	Ip                  string
	Port                int
	PublicUrl           string
	Locations           []*DiskLocation
	dataCenter          string 					//optional informaton, overwriting master setting if exists
	rack                string 					//optional information, overwriting master setting if exists
	connected           bool
	NeedleMapType       NeedleMapType
	NewVolumesChan      chan master_pb.VolumeShortInformationMessage
	DeletedVolumesChan  chan master_pb.VolumeShortInformationMessage
	NewEcShardsChan     chan master_pb.VolumeEcShardInformationMessage
	DeletedEcShardsChan chan master_pb.VolumeEcShardInformationMessage
}
```

