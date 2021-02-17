# DiskLocation

```go
type DiskLocation struct {
	Directory      string				// 目录，进程启动时指定
	MaxVolumeCount int				// 目录下最多volume的个数，进程启动时指定
	volumes        map[needle.VolumeId]*Volume
	sync.RWMutex

	// erasure coding
	ecVolumes     map[needle.VolumeId]*erasure_coding.EcVolume
	ecVolumesLock sync.RWMutex
}
```

