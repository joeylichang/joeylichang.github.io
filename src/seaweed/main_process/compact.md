# Compact

seaweed的数据删除是标记删除，既在索引文件中设置一个tombstone标记，mater通过心跳收集volume信息可以获取删除数据对volume的占比，当占比找过一个闸值（默认0.3）是就会执行一次压缩。seaweed的压缩原理比较简单，原地扫描copy一份数据（该过程将删除数据过滤掉）。

## 流程

1. master启动成功之后调用StartRefreshWritableVolumes。
2. StartRefreshWritableVolumes周期（15min）调用Vacuum去check volume删除数据的闸值。
3. Vacuum先check之前的compact是否执行完，否则直接返回。
4. 扫描collect的VolumeLayout，执行vacuumOneVolumeLayout。
5. vacuumOneVolumeLayout遍历VolumeLayout的locationList（dn list）。
   1. 执行batchVacuumVolumeCheck，如果达到闸值执行batchVacuumVolumeCompact
   2. 如果batchVacuumVolumeCompact执行成功，继续执行batchVacuumVolumeCommit
   3. 否则执行batchVacuumVolumeCleanup。

##源码

下面重点看一下batchVacuumVolumeCheck、batchVacuumVolumeCompact、batchVacuumVolumeCommit、batchVacuumVolumeCleanup的逻辑和volume_server的交互。

### batchVacuumVolumeCheck

###### master

```go
func batchVacuumVolumeCheck(grpcDialOption grpc.DialOption, vl *VolumeLayout, vid needle.VolumeId, locationlist *VolumeLocationList, garbageThreshold float64) bool {
	ch := make(chan bool, locationlist.Length())
	for index, dn := range locationlist.list {
		go func(index int, url string, vid needle.VolumeId) {
		
      			// 调用grpc向volume_server发送请求
			err := operation.WithVolumeServerClient(url, grpcDialOption, func(volumeServerClient volume_server_pb.VolumeServerClient) error {
				resp, err := volumeServerClient.VacuumVolumeCheck(context.Background(), &volume_server_pb.VacuumVolumeCheckRequest{
					VolumeId: uint32(vid),
				})
				if err != nil {
					ch <- false
					return err
				}
				isNeeded := resp.GarbageRatio > garbageThreshold
        			
				// 通过管道返回结果
				ch <- isNeeded
				return nil
			})
			if err != nil {
				glog.V(0).Infof("Checking vacuuming %d on %s: %v", vid, url, err)
			}
		}(index, dn.Url(), vid)
	}
	isCheckSuccess := true
  
  	// 通过for循环可以看出来，必须所有的dn都表示需要超过闸值才能进行compact
	for _ = range locationlist.list {
		select {
		case canVacuum := <-ch:
			isCheckSuccess = isCheckSuccess && canVacuum
		case <-time.After(30 * time.Minute):
			isCheckSuccess = false
			break
		}
	}
	return isCheckSuccess
}
```

compact、commit、cleanup的master侧代码类似，不再赘述。

###### volume_server

volume_server侧batchVacuumVolumeCheck的逻辑只是单纯的返回garbageRatio（删除数据占比）。



### batchVacuumVolumeCompact

###### volume_server

```go
unc (v *Volume) Compact(preallocate int64, compactionBytePerSecond int64) error {

  	// 如果在allocvolume是指定了文件缓存则不进行compact
	if v.MemoryMapMaxSizeMb == 0 { //it makes no sense to compact in memory
		glog.V(3).Infof("Compacting volume %d ...", v.Id)
		v.isCompacting = true
		defer func() {
			v.isCompacting = false
		}()

		filePath := v.FileName()
		v.lastCompactIndexOffset = v.IndexFileSize()
		v.lastCompactRevision = v.SuperBlock.CompactionRevision
		glog.V(3).Infof("creating copies for volume %d ,last offset %d...", v.Id, v.lastCompactIndexOffset)
    
    		// 创建数据备份文件（.cpd）和索引备份文件（.cpx），并且逐条读取数据写入备份文件
    		// 扫描数据文件时使用scanner结构，其中使用了btree作为中间文件的索引结构关联备份索引文件？为啥呢
    		// 还会根据参数控制compact的速度
		return v.copyDataAndGenerateIndexFile(filePath+".cpd", filePath+".cpx", preallocate, compactionBytePerSecond)
	} else {
		return nil
	}
}
```



### batchVacuumVolumeCommit

###### volume_server

```go
func (v *Volume) CommitCompact() error {
  	// 判断是否有缓存
	if v.MemoryMapMaxSizeMb == 0 { //it makes no sense to compact in memory
		glog.V(0).Infof("Committing volume %d vacuuming...", v.Id)

		v.isCompacting = true
		defer func() {
			v.isCompacting = false
		}()

		v.dataFileAccessLock.Lock()
		defer v.dataFileAccessLock.Unlock()

		glog.V(3).Infof("Got volume %d committing lock...", v.Id)
		v.nm.Close()
		if err := v.dataFile.Close(); err != nil {
			glog.V(0).Infof("fail to close volume %d", v.Id)
		}
		v.dataFile = nil
		stats.VolumeServerVolumeCounter.WithLabelValues(v.Collection, "volume").Dec()

		var e error
    		/* 做一些完整性校验（文件大小是否可以包含证书和索引的大小）
     		 * 追加diff，虽然在batchVacuumVolumeCompact阶段会将volume从可写集合摘除（master侧逻辑）
    		 * 但是摘除期间还是有时间diff，期间可能有数据写入，所以要追加一下diff
     		 */
		if e = v.makeupDiff(v.FileName()+".cpd", v.FileName()+".cpx", v.FileName()+".dat", v.FileName()+".idx"); e != nil {
			glog.V(0).Infof("makeupDiff in CommitCompact volume %d failed %v", v.Id, e)
			e = os.Remove(v.FileName() + ".cpd")
			if e != nil {
				return e
			}
			e = os.Remove(v.FileName() + ".cpx")
			if e != nil {
				return e
			}
		} else {
		
      			// rename 本分文件为线上使用文件
			var e error
			if e = os.Rename(v.FileName()+".cpd", v.FileName()+".dat"); e != nil {
				return fmt.Errorf("rename %s: %v", v.FileName()+".cpd", e)
			}
			if e = os.Rename(v.FileName()+".cpx", v.FileName()+".idx"); e != nil {
				return fmt.Errorf("rename %s: %v", v.FileName()+".cpx", e)
			}
		}

    		// 删除索引文件btree（bdb）
		os.RemoveAll(v.FileName() + ".ldb")
		os.RemoveAll(v.FileName() + ".bdb")

    		// 加载文件
		glog.V(3).Infof("Loading volume %d commit file...", v.Id)
		if e = v.load(true, false, v.needleMapKind, 0); e != nil {
			return e
		}
	}
	return nil
}
```

##### 问题

1. 为什么扫描的时候使用btree建立做引。
2. commit中makeupDiff使用的是idx索引文件，是不是leveldb不支持compact或者压缩之后也变成idx文件？

### batchVacuumVolumeCleanup

###### volume_server

```go
func (v *Volume) cleanupCompact() error {
	glog.V(0).Infof("Cleaning up volume %d vacuuming...", v.Id)

  	// 清理失败的备份文件
	e1 := os.Remove(v.FileName() + ".cpd")
	e2 := os.Remove(v.FileName() + ".cpx")
	if e1 != nil {
		return e1
	}
	if e2 != nil {
		return e2
	}
	return nil
}
```
