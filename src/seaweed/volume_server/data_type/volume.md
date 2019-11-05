# Volume

```go
type Volume struct {
	Id                 needle.VolumeId
	dir                string
	Collection         string			// collect 名称，master下发
	dataFile           *os.File			// .dat文件
	nm                 NeedleMapper			// 索引文件抽象
	needleMapKind      NeedleMapType		// 索引文件类型
	readOnly           bool
	MemoryMapMaxSizeMb uint32			// 数据文件的内存缓存大小，注意：如果设置了将不进行compact，master创建volume时指定

	SuperBlock					// volume元数据，见后面介绍

	dataFileAccessLock    sync.Mutex
	lastModifiedTsSeconds uint64 //unix time in seconds
	lastAppendAtNs        uint64 //unix time in nanoseconds

	lastCompactIndexOffset uint64			// 压缩的进度表示
	lastCompactRevision    uint16			// 压缩的版本号，主从同步等情况使用

	isCompacting bool				// 是否正在进行压缩的标志
}
```

