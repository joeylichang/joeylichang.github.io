# SuperBlock

```go
/*
* Super block currently has 8 bytes allocated for each volume.
* Byte 0: version, 1 or 2
* Byte 1: Replica Placement strategy, 000, 001, 002, 010, etc
* Byte 2 and byte 3: Time to live. See TTL for definition
* Byte 4 and byte 5: The number of times the volume has been compacted.
* Rest bytes: Reserved
 */

type SuperBlock struct {
	version            needle.Version
	ReplicaPlacement   *ReplicaPlacement
	Ttl                *needle.TTL
	CompactionRevision uint16
	Extra              *master_pb.SuperBlockExtra		// ec使用
	extraSize          uint16
}
```

