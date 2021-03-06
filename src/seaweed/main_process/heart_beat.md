# Heartbeat

seaweed中master是被动收集心跳，主要有以下几点：

1. volume_server 根据配置（默认5s）汇报一次心跳，master根据收到的心跳更新dn的信息。
2. 根据master的Heartbeat源码，volume_server与master之间是长连接，一旦链接断开相应的元信息（rack、dc的volume个数等）进行相应的扣减。

问题：

1. 在源码中并没有周期性的check dn的过期时间（dn中LastSeen字段），如果链接故障（因为是长连接可能心跳丢失或者其他原因master并未收到，或者dn故障了master并未收到链接断开的网络分节）会导致master做一些误判（volume_server pick不准确等 ==> 这可能是volume在创建时会有一个放大的原因，master的grow_option中有介绍）。
2. 从源码中看函数StartRefreshWritableVolumes，像是包含周期ckeck dn的逻辑，但是里面之后check full 和 garbage的逻辑。

## 消息

```go
type Heartbeat struct {
	Ip             string                      
	Port           uint32                      
	PublicUrl      string                      
	MaxVolumeCount uint32                      
	MaxFileKey     uint64                      
	DataCenter     string                      
	Rack           string                      
	AdminPort      uint32                      
	Volumes        []*VolumeInformationMessage 
  
	// delta volumes
	NewVolumes     []*VolumeShortInformationMessage 
	DeletedVolumes []*VolumeShortInformationMessage 
	HasNoVolumes   bool   
}

type VolumeShortInformationMessage struct {
	Id               uint32 
	Collection       string 
	ReplicaPlacement uint32 
	Version          uint32 
	Ttl              uint32 
}

type HeartbeatResponse struct {
	VolumeSizeLimit        uint64 
	Leader                 string 
	MetricsAddress         string 
	MetricsIntervalSeconds uint32 
}
```



## 源码

###### master侧源码

```go
func (ms *MasterServer) SendHeartbeat(stream master_pb.Seaweed_SendHeartbeatServer) error {
	var dn *topology.DataNode
	t := ms.Topo

  	// 链接断开的逻辑
	defer func() {
		if dn != nil {

      			glog.V(0).Infof("unregister disconnected volume server %s:%d", dn.Ip, dn.Port)
		
      			/* 设置VolumeLayout 为 Unavailable（VolumeLayout中可能有多个dn，只取出一个dn）
       			 * 更新tupu中各节点信息并且从拓扑中摘除 */
			t.UnRegisterDataNode(dn)

			message := &master_pb.VolumeLocation{
				Url:       dn.Url(),
				PublicUrl: dn.PublicUrl,
			}
			for _, v := range dn.GetVolumes() {
				message.DeletedVids = append(message.DeletedVids, uint32(v.Id))
			}

      			/* 一个类似client monitor 的功能，像信息发送给client */
			if len(message.DeletedVids) > 0 {
				ms.clientChansLock.RLock()
				for _, ch := range ms.clientChans {
					ch <- message
				}
				ms.clientChansLock.RUnlock()
			}

		}
	}()

	for {
    		// 链接正常，持续从长连接中读出数据
		heartbeat, err := stream.Recv()
		// 更新全局最大的key
		t.Sequence.SetMax(heartbeat.MaxFileKey)

    		// 表示链接第一次建立
		if dn == nil {
			// 从配置中读取，配置可以是空
			dcName, rackName := t.Configuration.Locate(heartbeat.Ip, heartbeat.DataCenter, heartbeat.Rack)
      			// 从拓扑中获取dn，如果没有就会常见，其中会更新dn的LastSeen
			dc := t.GetOrCreateDataCenter(dcName)
			rack := dc.GetOrCreateRack(rackName)
			dn = rack.GetOrCreateDataNode(heartbeat.Ip,
				int(heartbeat.Port), heartbeat.PublicUrl,
				int64(heartbeat.MaxVolumeCount))
			glog.V(0).Infof("added volume server %v:%d", heartbeat.GetIp(), heartbeat.GetPort())
			if err := stream.Send(&master_pb.HeartbeatResponse{
				VolumeSizeLimit: uint64(ms.option.VolumeSizeLimitMB) * 1024 * 1024,
			}); err != nil {
				glog.Warningf("SendHeartbeat.Send volume size to %s:%d %v", dn.Ip, dn.Port, err)
				return err
			}
		}

		glog.V(4).Infof("master received heartbeat %s", heartbeat.String())
		message := &master_pb.VolumeLocation{
			Url:       dn.Url(),
			PublicUrl: dn.PublicUrl,
		}
    		// 新增和删除vid更新topu
		if len(heartbeat.NewVolumes) > 0 || len(heartbeat.DeletedVolumes) > 0 {
			// process delta volume ids if exists for fast volume id updates
			for _, volInfo := range heartbeat.NewVolumes {
				message.NewVids = append(message.NewVids, volInfo.Id)
			}
			for _, volInfo := range heartbeat.DeletedVolumes {
				message.DeletedVids = append(message.DeletedVids, volInfo.Id)
			}
			// 更新dn 的 VolumeInfo 并且会更新VolumeLayout
			t.IncrementalSyncDataNodeRegistration(heartbeat.NewVolumes, heartbeat.DeletedVolumes, dn)
		}

    		// 更新volume_server 原本的volume信息
		if len(heartbeat.Volumes) > 0 || heartbeat.HasNoVolumes {
			// 根据heartbeat.Volumes 获取到volume的变化
			newVolumes, deletedVolumes := t.SyncDataNodeRegistration(heartbeat.Volumes, dn)

			for _, v := range newVolumes {
				glog.V(0).Infof("master see new volume %d from %s", uint32(v.Id), dn.Url())
				message.NewVids = append(message.NewVids, uint32(v.Id))
			}
			for _, v := range deletedVolumes {
				glog.V(0).Infof("master see deleted volume %d from %s", uint32(v.Id), dn.Url())
				message.DeletedVids = append(message.DeletedVids, uint32(v.Id))
			}
		}

   		// 通知订阅的客户端
		if len(message.NewVids) > 0 || len(message.DeletedVids) > 0 {
			ms.clientChansLock.RLock()
			for host, ch := range ms.clientChans {
				glog.V(0).Infof("master send to %s: %s", host, message.String())
				ch <- message
			}
			ms.clientChansLock.RUnlock()
		}

		// tell the volume servers about the leader
		newLeader, err := t.Leader()
		if err != nil {
			glog.Warningf("SendHeartbeat find leader: %v", err)
			return err
		}
		if err := stream.Send(&master_pb.HeartbeatResponse{
			Leader:                 newLeader,
			MetricsAddress:         ms.option.MetricsAddress,
			MetricsIntervalSeconds: uint32(ms.option.MetricsIntervalSec),
		}); err != nil {
			glog.Warningf("SendHeartbeat.Send response to to %s:%d %v", dn.Ip, dn.Port, err)
			return err
		}
	}
}
```

###### volume_server 侧源码

volume_server初始化完毕后，有一个协程周期（默认5s）上报心跳。

```go
func (vs *VolumeServer) heartbeat() {

	glog.V(0).Infof("Volume server start with seed master nodes: %v", vs.SeedMasterNodes)
	vs.store.SetDataCenter(vs.dataCenter)
	vs.store.SetRack(vs.rack)

	grpcDialOption := security.LoadClientTLS(viper.Sub("grpc"), "volume")

	var err error
	var newLeader string
	for {
		for _, master := range vs.SeedMasterNodes {
			if newLeader != "" {
				master = newLeader
			}
			masterGrpcAddress, parseErr := util.ParseServerToGrpcAddress(master)
			if parseErr != nil {
				glog.V(0).Infof("failed to parse master grpc %v: %v", masterGrpcAddress, parseErr)
				continue
			}
			vs.store.MasterAddress = master
			
      			// 收集详细信息
			newLeader, err = vs.doHeartbeat(context.Background(), master, masterGrpcAddress, grpcDialOption, time.Duration(vs.pulseSeconds)*time.Second)
			if err != nil {
				glog.V(0).Infof("heartbeat error: %v", err)
				
        			// 周期执行
				time.Sleep(time.Duration(vs.pulseSeconds) * time.Second)
				newLeader = ""
				vs.store.MasterAddress = ""
			}
		}
	}
}
```

```go
func (vs *VolumeServer) doHeartbeat(ctx context.Context, masterNode, masterGrpcAddress string, grpcDialOption grpc.DialOption, sleepInterval time.Duration) (newLeader string, err error) {

	grpcConection, err := util.GrpcDial(ctx, masterGrpcAddress, grpcDialOption)
	if err != nil {
		return "", fmt.Errorf("fail to dial %s : %v", masterNode, err)
	}
	defer grpcConection.Close()

	client := master_pb.NewSeaweedClient(grpcConection)
  
  	// 建立长连接
	stream, err := client.SendHeartbeat(ctx)
	if err != nil {
		glog.V(0).Infof("SendHeartbeat to %s: %v", masterNode, err)
		return "", err
	}
	glog.V(0).Infof("Heartbeat to: %v", masterNode)
	vs.currentMaster = masterNode

	doneChan := make(chan error, 1)

  	// 接收心跳的返回包
	go func() {
		for {
			in, err := stream.Recv()
			if err != nil {
				doneChan <- err
				return
			}
			if in.GetVolumeSizeLimit() != 0 {
				vs.store.SetVolumeSizeLimit(in.GetVolumeSizeLimit())
			}
			if in.GetLeader() != "" && masterNode != in.GetLeader() && !isSameIP(in.GetLeader(), masterNode) {
				glog.V(0).Infof("Volume Server found a new master newLeader: %v instead of %v", in.GetLeader(), masterNode)
				newLeader = in.GetLeader()
        
        			// 主master变更，需要重连，退出
				doneChan <- nil
				return
			}
			if in.GetMetricsAddress() != "" && vs.MetricsAddress != in.GetMetricsAddress() {
				vs.MetricsAddress = in.GetMetricsAddress()
				vs.MetricsIntervalSec = int(in.GetMetricsIntervalSeconds())
			}
		}
	}()

  	// 第一次发送数据
	if err = stream.Send(vs.store.CollectHeartbeat()); err != nil {
		glog.V(0).Infof("Volume Server Failed to talk with master %s: %v", masterNode, err)
		return "", err
	}

	volumeTickChan := time.Tick(sleepInterval)

	for {
		select {
		
    		// 新建volume通过tology的管道通知发送
		case volumeMessage := <-vs.store.NewVolumesChan:
			deltaBeat := &master_pb.Heartbeat{
				NewVolumes: []*master_pb.VolumeShortInformationMessage{
					&volumeMessage,
				},
			}
			glog.V(1).Infof("volume server %s:%d adds volume %d", vs.store.Ip, vs.store.Port, volumeMessage.Id)
			if err = stream.Send(deltaBeat); err != nil {
				glog.V(0).Infof("Volume Server Failed to update to master %s: %v", masterNode, err)
				return "", err
			}
			
    		// 删除volume通过tology的管道通知发送
		case volumeMessage := <-vs.store.DeletedVolumesChan:
			deltaBeat := &master_pb.Heartbeat{
				DeletedVolumes: []*master_pb.VolumeShortInformationMessage{
					&volumeMessage,
				},
			}
			glog.V(1).Infof("volume server %s:%d deletes volume %d", vs.store.Ip, vs.store.Port, volumeMessage.Id)
			if err = stream.Send(deltaBeat); err != nil {
				glog.V(0).Infof("Volume Server Failed to update to master %s: %v", masterNode, err)
				return "", err
			}
			
    		// 周期发送
		case <-volumeTickChan:
			glog.V(4).Infof("volume server %s:%d heartbeat", vs.store.Ip, vs.store.Port)
			if err = stream.Send(vs.store.CollectHeartbeat()); err != nil {
				glog.V(0).Infof("Volume Server Failed to talk with master %s: %v", masterNode, err)
				return "", err
			}
		case err = <-doneChan:
			return
		}
	}
}
```



