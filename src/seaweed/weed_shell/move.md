# Move

Move的作用是在线迁移一个volume。

#### 流程

1. 向目标dn发送请求，让其向源dn发起copy请求，copy数据文件和索引文件，copy期间会记录最后的时间戳。
2. 目标节点收完数据之后会mount volume。
3. 目标节点向源节点发起TailReceiver请求，既在步骤一的时间戳之后的请求会追加给目标节点。
4. TailReceiver的逻辑中有一个时间窗口，既追加完请求之后再这个时间窗口没有更新才算追加成功。
5. 向源dn发送umount请求。
6. 向源dn发送delete volume的请求。

#### 问题

1. 步骤2中，注释说目标dn在此处介绍说会通知master标记该节点为只读，但是逻辑中并没有该部分逻辑（如果写入流量较大可能有问题）。
2. 步骤5中，注释说会通知master节点擦除volume的只读，但是逻辑中并没有该部分逻辑。
3. 如果copy、tail阶段有异常退出之后，并没有删除目标节点文件的逻辑，这可能是有问题的。

#### 源码

如上所述代码的核心罗成是copy、tail和delete（从内存数据删除和删除文件比较简单不再赘述）是核心逻辑，下面看一下这部分源码：

###### copy逻辑

```go
func (vs *VolumeServer) VolumeCopy(ctx context.Context, req *volume_server_pb.VolumeCopyRequest) (*volume_server_pb.VolumeCopyResponse, error) {
	/* 略 */
	var volFileInfoResp *volume_server_pb.ReadVolumeFileStatusResponse
	var volumeFileName, idxFileName, datFileName string
	err := operation.WithVolumeServerClient(req.SourceDataNode, vs.grpcDialOption, func(client volume_server_pb.VolumeServerClient) error {
		var err error
    
    // 先获取.dat、.idx文件的stat信息
		volFileInfoResp, err = client.ReadVolumeFileStatus(ctx,
			&volume_server_pb.ReadVolumeFileStatusRequest{
				VolumeId: req.VolumeId,
			})
		if nil != err {
			return fmt.Errorf("read volume file status failed, %v", err)
		}

		volumeFileName = storage.VolumeFileName(location.Directory, volFileInfoResp.Collection, int(req.VolumeId))

		// 貌似只支持idx的copy，不支持leveldb索引的copy
		if err := vs.doCopyFile(ctx, client, false, req.Collection, req.VolumeId, volFileInfoResp.CompactionRevision, volFileInfoResp.IdxFileSize, volumeFileName, ".idx", false); err != nil {
			return err
		}

		if err := vs.doCopyFile(ctx, client, false, req.Collection, req.VolumeId, volFileInfoResp.CompactionRevision, volFileInfoResp.DatFileSize, volumeFileName, ".dat", false); err != nil {
			return err
		}

		return nil
	})

	idxFileName = volumeFileName + ".idx"
	datFileName = volumeFileName + ".dat"

	if err != nil && volumeFileName != "" {
		if idxFileName != "" {
			os.Remove(idxFileName)
		}
		if datFileName != "" {
			os.Remove(datFileName)
		}
		return nil, err
	}

  // 校验和之前获取的文件state信息是否相符
	if err = checkCopyFiles(volFileInfoResp, idxFileName, datFileName); err != nil { // added by panyc16
		return nil, err
	}

	// mount the volume
	err = vs.store.MountVolume(needle.VolumeId(req.VolumeId))
	if err != nil {
		return nil, fmt.Errorf("failed to mount volume %d: %v", req.VolumeId, err)
	}

	return &volume_server_pb.VolumeCopyResponse{
		LastAppendAtNs: volFileInfoResp.DatFileTimestampSeconds * uint64(time.Second),
	}, err
}
```

doCopyFile做了两件事：1.是想源dn发起copy请求，2. 将数据写到指定位置。下面看一下源dn的逻辑：

```go
func (vs *VolumeServer) CopyFile(req *volume_server_pb.CopyFileRequest, stream volume_server_pb.VolumeServer_CopyFileServer) error {

	var fileName string
	{
		baseFileName := erasure_coding.EcShardBaseFileName(req.Collection, int(req.VolumeId)) + req.Ext
		for _, location := range vs.store.Locations {
			tName := path.Join(location.Directory, baseFileName)
			if util.FileExists(tName) {
				fileName = tName
			}
		}
		if fileName == "" {
			return fmt.Errorf("CopyFile not found ec volume id %d", req.VolumeId)
		}
	}

  // 之前目标dn发起state请求时候文件大小，后面通过tail追加
	bytesToRead := int64(req.StopOffset)

	file, err := os.Open(fileName)
	if err != nil {
		return err
	}
	defer file.Close()

	buffer := make([]byte, BufferSizeLimit)

	for bytesToRead > 0 {
		bytesread, err := file.Read(buffer)
		if err != nil {
			if err != io.EOF {
				return err
			}
			break
		}

		if int64(bytesread) > bytesToRead {
			bytesread = int(bytesToRead)
		}
    // 发送数据
		err = stream.Send(&volume_server_pb.CopyFileResponse{
			FileContent: buffer[:bytesread],
		})
		if err != nil {
			return err
		}

		bytesToRead -= int64(bytesread)

	}

	return nil
}
```



###### tail逻辑

tail的逻辑设计两个请求：VolumeTailReceiver、VolumeTailSender。

1. 向目标dn发送VolumeTailReceiver请求参数中包括两个参数值得注意，既SinceNs（之前同步的最后一条数据的追加时间）、IdleTimeoutSeconds（追加玩数据之后等这么长时间没有请求才认为同步成功）。
2. 目标dn向源dn发送VolumeTailSender请求，让其根据要求发送数据。

可能是IdleTimeoutSeconds的设计，让作者省去了vid设置、擦除只读的流程（上面提出来的问题部分），但是这个设计显然不严谨。

VolumeTailReceiver只是接受数据、拼装needle、追加volume逻辑相对简单不再赘述，重点看一下VolumeTailSender逻辑：

```go
func (vs *VolumeServer) VolumeTailSender(req *volume_server_pb.VolumeTailSenderRequest, stream volume_server_pb.VolumeServer_VolumeTailSenderServer) error {
	/* 略 */
	lastTimestampNs := req.SinceNs
	drainingSeconds := req.IdleTimeoutSeconds

	for {
    // 根据lastTimestampNs 二分查找发送需要的数据
		lastProcessedTimestampNs, err := sendNeedlesSince(stream, v, lastTimestampNs)
		if err != nil {
			glog.Infof("sendNeedlesSince: %v", err)
			return fmt.Errorf("streamFollow: %v", err)
		}
		time.Sleep(2 * time.Second)

    // 为0表示在同步完数据等待期间有数据发送需要重新开始
		if req.IdleTimeoutSeconds == 0 {
			lastTimestampNs = lastProcessedTimestampNs
			continue
		}
    // 表示最后一条数据在之前的copy已经发送了，没有新数据drainingSeconds进行记时
    // 前面sleep一次是2s，所以相当于等了drainingSeconds * 2
		if lastProcessedTimestampNs == lastTimestampNs {
			drainingSeconds--
			if drainingSeconds <= 0 {
				return nil
			}
			glog.V(1).Infof("tailing volume %d drains requests with %d seconds remaining", v.Id, drainingSeconds)
		} else {
      // 到这里说明在copy只有有数据追加，需要更新一下时间重新计数
			lastTimestampNs = lastProcessedTimestampNs
			drainingSeconds = req.IdleTimeoutSeconds
			glog.V(1).Infof("tailing volume %d resets draining wait time to %d seconds", v.Id, drainingSeconds)
		}

	}

}
```

