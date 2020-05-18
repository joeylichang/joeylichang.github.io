# Assign

Assign请求的作用是想master申请fid，有两种接口 既http，grpc，其逻辑是相同（以下源码以grpc接口开始讲解）。

## 参数

```go
type AssignRequest struct {
	Count              uint64 		// 申请的fid个数
	Replication        string 		// 副本策略
	Collection         string 		// collection名，可以为空
	Ttl                string 		// 存活时长
	DataCenter         string 		
	Rack               string 
	DataNode           string 
	MemoryMapMaxSizeMb uint32 		// volume在内存的缓存大小，如果不为0，则不会压缩（建议为0）
}

type AssignResponse struct {
	Fid       string 			// 申请多个fid也只会返回一个
	Url       string 			// dn的url，只有一个，主dn保证强一致性
	PublicUrl string 
	Count     uint64 			// 申请成功的个数
	Error     string 
	Auth      string 			// 设置了安全配置，则需要这个选项（不做重点介绍）
}
```

注意：申请多个fid时，只返回一个，fid由三部分组成，vid + 内存递增id + cookie，多个fid回退即可获得

## 流程

1. 根据请求参数生成[VolumeGrowOption](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/seaweed/master/tupo/volume_grow_option.md)（选取volume，或者创建volume的必要参数）。
2. 根据参数要求选取volume，如果没有相应volume，master根据策略创建volume。
3. 创建之后根据参数选取volumeid、fid，生成最终的Fid，拼装AssignResponse返回。

## 源码

###### master侧源码

```go
func (ms *MasterServer) Assign(ctx context.Context, req *master_pb.AssignRequest) (*master_pb.AssignResponse, error) {

	/* 省略参数校验 */
	option := &topology.VolumeGrowOption{
		Collection:         req.Collection,
		ReplicaPlacement:   replicaPlacement,
		Ttl:                ttl,
		Prealloacte:        ms.preallocateSize,
		DataCenter:         req.DataCenter,
		Rack:               req.Rack,
		DataNode:           req.DataNode,
		MemoryMapMaxSizeMb: req.MemoryMapMaxSizeMb,
	}
	
  /*
   * HasWritableVolume逻辑：
   * 1. 根据option.Collection获取collection -> map(id, volumeLayout)的map
   * 2. 根据id = option.ReplicaPlacement + option.Ttl -> volumeLayout 获取collect、rp、ttl符合要求的volume（Collection、volumeLayout等元数据组织见master拓扑数据部分介绍）
   * 3. 期间如果没有collection，或者volumeLayout，则创建新的。
   */
	if !ms.Topo.HasWritableVolume(option) {
		if ms.Topo.FreeSpace() <= 0 {
			return nil, fmt.Errorf("No free volumes left!")
		}
		ms.vgLock.Lock()
    
  		// 加锁需要重新check
		if !ms.Topo.HasWritableVolume(option) {
      			// 没有符合要求的volume，此时需要根据要求创建新的volume
			if _, err = ms.vg.AutomaticGrowByType(option, ms.grpcDialOption, ms.Topo); err != nil {
				ms.vgLock.Unlock()
				return nil, fmt.Errorf("Cannot grow volume group! %v", err)
			}
		}
		ms.vgLock.Unlock()
	}
  
  	// 注意count是申请成功的数量，但是fid只返回了一个，原因见上面参数部分
	fid, count, dn, err := ms.Topo.PickForWrite(req.Count, option)
	if err != nil {
		return nil, fmt.Errorf("%v", err)
	}

	return &master_pb.AssignResponse{
		Fid:       fid,
		Url:       dn.Url(),
		PublicUrl: dn.PublicUrl,
		Count:     count,
		Auth:      string(security.GenJwt(ms.guard.SigningKey, ms.guard.ExpiresAfterSec, fid)),
	}, nil
}
```

下面重点看一下AutomaticGrowByType逻辑，调用链关系：AutomaticGrowByType -> GrowByCountAndType -> findAndGrow -> findEmptySlotsForOneVolume / grow，直接看一下findEmptySlotsForOneVolume 和 grow的逻辑：

```go
func (vg *VolumeGrowth) findEmptySlotsForOneVolume(topo *Topology, option *VolumeGrowOption) (servers []*DataNode, err error) {
	/* 根据ReplicaPlacement策略，从dc、rack、dn主从选取，每层的选取的逻辑都是RandomlyPickNodes
	 * RandomlyPickNodes中参数func是校验的逻辑，既判断节点是否满足option内的要求
	 * RandomlyPickNodes逻辑：
	 * 1. 扫描同级别的所有节点，判断是否通过func的校验，不通过的抛弃
	 * 2. 在通过的备选集中第一个节点为main，其他的节点选取目标个
	 *    2.1 在m个选取n个，按顺序扫描m，如果i<n，放入备选集
	 *    2.2 如果i >= n，替换第rand(i + 1)的备选集
	 *    2.3 最终的备选集为other返回
	 */
	rp := option.ReplicaPlacement
	mainDataCenter, otherDataCenters, dc_err := topo.RandomlyPickNodes(rp.DiffDataCenterCount+1, func(node Node) error {
    		// 1. 必须是dc，如果指定dc，则必须为指定的值
		if option.DataCenter != "" && node.IsDataCenter() && node.Id() != NodeId(option.DataCenter) {
			return fmt.Errorf("Not matching preferred data center:%s", option.DataCenter)
		}
    		// 2. dc的子节点既rack必须符合rp的策略要求
		if len(node.Children()) < rp.DiffRackCount+1 {
			return fmt.Errorf("Only has %d racks, not enough for %d.", len(node.Children()), rp.DiffRackCount+1)
		}
    		// 3. dc容量必须满足rp策略
		if node.FreeSpace() < int64(rp.DiffRackCount+rp.SameRackCount+1) {
			return fmt.Errorf("Free:%d < Expected:%d", node.FreeSpace(), rp.DiffRackCount+rp.SameRackCount+1)
		}
    
    		// 4. 计算当前dc下，rack和dn有容量的节点是否满足rp
		possibleRacksCount := 0
		for _, rack := range node.Children() {
			possibleDataNodesCount := 0
			for _, n := range rack.Children() {
				if n.FreeSpace() >= 1 {
					possibleDataNodesCount++
				}
			}
			if possibleDataNodesCount >= rp.SameRackCount+1 {
				possibleRacksCount++
			}
		}
		if possibleRacksCount < rp.DiffRackCount+1 {
			return fmt.Errorf("Only has %d racks with more than %d free data nodes, not enough for %d.", possibleRacksCount, rp.SameRackCount+1, rp.DiffRackCount+1)
		}
		return nil
	})
	if dc_err != nil {
		return nil, dc_err
	}

	//find main rack and other racks
	mainRack, otherRacks, rackErr := mainDataCenter.(*DataCenter).RandomlyPickNodes(rp.DiffRackCount+1, func(node Node) error {
		/* 略 */
	})
	if rackErr != nil {
		return nil, rackErr
	}

	//find main rack and other racks
	mainServer, otherServers, serverErr := mainRack.(*Rack).RandomlyPickNodes(rp.SameRackCount+1, func(node Node) error {
		/* 略 */
	})
	if serverErr != nil {
		return nil, serverErr
	}

  	// 将同rack下选出来的dn加到结果中
	servers = append(servers, mainServer.(*DataNode))
	for _, server := range otherServers {
		servers = append(servers, server.(*DataNode))
	}
  
  	/* 在otherRacks中随机选择一个server，这里面随机用的rand.Int63n(rack.FreeSpace())
  	 * 随机性用了FreeSpace，经过一定长时间的悬着之后每个节点的容量将是均衡的
   	 */
	for _, rack := range otherRacks {
		r := rand.Int63n(rack.FreeSpace())
		if server, e := rack.ReserveOneVolume(r); e == nil {
			servers = append(servers, server)
		} else {
			return servers, e
		}
	}
	
  	/* ReserveOneVolume 进行递归计算dn容量，逻辑同上 */
	for _, datacenter := range otherDataCenters {
		r := rand.Int63n(datacenter.FreeSpace())
		if server, e := datacenter.ReserveOneVolume(r); e == nil {
			servers = append(servers, server)
		} else {
			return servers, e
		}
	}
	return
}
```

上面已经选好了dn，下面看一下grow的逻辑：

```go
func (vg *VolumeGrowth) grow(grpcDialOption grpc.DialOption, topo *Topology, vid needle.VolumeId, option *VolumeGrowOption, servers ...*DataNode) error {
	for _, server := range servers {
    		// 调用grpc 与 volume_server交互
		if err := AllocateVolume(server, grpcDialOption, vid, option); err == nil {
			vi := storage.VolumeInfo{
				Id:               vid,
				Size:             0,
				Collection:       option.Collection,
				ReplicaPlacement: option.ReplicaPlacement,
				Ttl:              option.Ttl,
				Version:          needle.CurrentVersion,
			}
      			// 更新master本地拓扑信息
			server.AddOrUpdateVolume(vi)
			topo.RegisterVolumeLayout(vi, server)
			glog.V(0).Infoln("Created Volume", vid, "on", server.NodeImpl.String())
		} else {
			glog.V(0).Infoln("Failed to assign volume", vid, "to", servers, "error", err)
			return fmt.Errorf("Failed to assign %d: %v", vid, err)
		}
	}
	return nil
}
```

##### 问题

grow循环向所有的dn发送AllocateVolume请求，一旦有一个失败ReplicaPlacement将是不完整的，在seaweed系统本身的模块并没有这种一致性的校验，seaweed提供的解决方案是weed_shell（有一个fix.replica命令），建议周期性的执行，这种方案算是一种track方案，并不是一个完美方案。



###### volume_server侧源码

调用链：VolumeServer.AllocateVolume -> Store.AddVolume -> Store.addVolume

```go
func (s *Store) addVolume(vid needle.VolumeId, collection string, needleMapKind NeedleMapType, replicaPlacement *ReplicaPlacement, ttl *needle.TTL, preallocate int64, MemoryMapMaxSizeMb uint32) error {
	if s.findVolume(vid) != nil {
		return fmt.Errorf("Volume Id %d already exists!", vid)
	}
  
  	// 选择容量最少的磁盘
	if location := s.FindFreeLocation(); location != nil {
		glog.V(0).Infof("In dir %s adds volume:%v collection:%s replicaPlacement:%v ttl:%v",
			location.Directory, vid, collection, replicaPlacement, ttl)
		if volume, err := NewVolume(location.Directory, collection, vid, needleMapKind, replicaPlacement, ttl, preallocate, MemoryMapMaxSizeMb); err == nil {
      			// volume加到磁盘
			location.SetVolume(vid, volume)
			glog.V(0).Infof("add volume %d", vid)
			s.NewVolumesChan <- master_pb.VolumeShortInformationMessage{
				Id:               uint32(vid),
				Collection:       collection,
				ReplicaPlacement: uint32(replicaPlacement.Byte()),
				Version:          uint32(volume.Version()),
				Ttl:              ttl.ToUint32(),
			}
			return nil
		} else {
			return err
		}
	}
	return fmt.Errorf("No more free space left")
}
```



