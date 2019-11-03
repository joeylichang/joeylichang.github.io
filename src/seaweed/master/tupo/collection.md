# Collection

Collection是一层逻辑划分。Topology、DataCenter、Rack、DataNode这些属于物理划分。

```go
type Collection struct {
	Name                     string
	volumeSizeLimit          uint64
  storageType2VolumeLayout *util.ConcurrentReadMap   // string -> VolumeLayout(后续介绍)
                                                     // string = Replica策略 + ttl
}

func NewCollection(name string, volumeSizeLimit uint64) *Collection {
	c := &Collection{Name: name, volumeSizeLimit: volumeSizeLimit}
	c.storageType2VolumeLayout = util.NewConcurrentReadMap()
	return c
}
```

