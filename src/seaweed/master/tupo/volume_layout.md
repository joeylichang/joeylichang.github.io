# VolumeLayout

VolumeLayout 是vid 到dn的映射关系，读等操作直接在这里查找。

```go
// mapping from volume to its locations, inverted from server to volume
type VolumeLayout struct {
	rp               *storage.ReplicaPlacement
	ttl              *needle.TTL
	vid2location     map[needle.VolumeId]*VolumeLocationList
	writables        []needle.VolumeId        // transient array of writable volume id
	readonlyVolumes  map[needle.VolumeId]bool // transient set of readonly volumes
	oversizedVolumes map[needle.VolumeId]bool // set of oversized volumes
	volumeSizeLimit  uint64
	accessLock       sync.RWMutex
}
```



######  VolumeLocationList

```go
type VolumeLocationList struct {
	list []*DataNode
}
```

