# NeedleMapper

```go
// 心跳需要上报的信息
type mapMetric struct {
	DeletionCounter     uint32 `json:"DeletionCounter"`
	FileCounter         uint32 `json:"FileCounter"`
	DeletionByteCounter uint64 `json:"DeletionByteCounter"`
	FileByteCounter     uint64 `json:"FileByteCounter"`
	MaximumFileKey      uint64 `json:"MaxFileKey"`
}

type baseNeedleMapper struct {
	mapMetric

	indexFile           *os.File
	indexFileAccessLock sync.Mutex
}

// 纯内存map
type NeedleMap struct {
	baseNeedleMapper
	m needle_map.NeedleValueMap
}

// leveldb map
type LevelDbNeedleMap struct {
	baseNeedleMapper
	dbFileName string
	db         *leveldb.DB
}

// map存储的value
type NeedleValue struct {
	Key    NeedleId
	Offset Offset `comment:"Volume offset"` //since aligned to 8 bytes, range is 4G*8=32G
	Size   uint32 `comment:"Size of the data portion"`
}
```

