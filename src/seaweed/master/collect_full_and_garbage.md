# volume写满

master初始化函数中调用NewMasterServer，在NewMasterServer最后调用Topo.StartRefreshWritableVolumes做一些周期性的工作。源码如下：

```go
func (t *Topology) StartRefreshWritableVolumes(grpcDialOption grpc.DialOption, garbageThreshold float64, preallocate int64) {
	go func() {
		for {
			if t.IsLeader() {
				freshThreshHold := time.Now().Unix() - 3*t.pulse //3 times of sleep interval
				t.CollectDeadNodeAndFullVolumes(freshThreshHold, t.volumeSizeLimit)
			}
      			// pulse默认5ms，周期时间默认是5-10ms之间
			time.Sleep(time.Duration(float32(t.pulse*1e3)*(1+rand.Float32())) * time.Millisecond)
		}
	}()
  
  	// 当volume删除数据超过闸值之后，进行压缩这部分后面介绍
	go func(garbageThreshold float64) {
		c := time.Tick(15 * time.Minute)
		for _ = range c {
			if t.IsLeader() {
				t.Vacuum(grpcDialOption, garbageThreshold, preallocate)
			}
		}
	}(garbageThreshold)
  
  	// 上面CollectDeadNodeAndFullVolumes扫描全量的volume，发现满的会通过chan到这里
	go func() {
		for {
			select {
			case v := <-t.chanFullVolumes:
        			/* CollectDeadNodeAndFullVolumes扫描的是dn，返回的是VolumeInfo
        			 * SetVolumeCapacityFull 通过vid在volumeLayout中将vid所有的dn放入不可写的map中
        			 * 所以多个副本的vid扫描出一个就会被设置为不可写
        			 * 除了volumeLayout之外，还会更新父节点的VolumeCount信息
        			 */
				t.SetVolumeCapacityFull(v)
			}
		}
	}()
```



###### 注意

CollectDeadNodeAndFullVolumes(freshThreshHold, t.volumeSizeLimit)，中freshThreshHold参数并没有用，看freshThreshHold := time.Now().Unix() - 3*t.pulse （3 times of sleep interval） 的意思像是扫描所有dn判断其更新时间决定其是否存活（正如在HeartBeat中所说），但是作者在这里去掉这部分逻辑，可能是有一些隐情。
