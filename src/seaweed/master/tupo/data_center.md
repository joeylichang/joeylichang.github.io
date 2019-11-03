# DataCenter

```go
type DataCenter struct {
	NodeImpl
}

func NewDataCenter(id string) *DataCenter {
	dc := &DataCenter{}
	dc.id = NodeId(id)
	dc.nodeType = "DataCenter"
	dc.children = make(map[NodeId]Node)		
	dc.NodeImpl.value = dc
	return dc
}

func (dc *DataCenter) GetOrCreateRack(rackName string) *Rack {
	for _, c := range dc.Children() {
		rack := c.(*Rack)							// 子节点必须是rack
		if string(rack.Id()) == rackName {
			return rack
		}
	}
	rack := NewRack(rackName)
	dc.LinkChildNode(rack)					// NodeImpl的方法，主要是children、SetParent完成父子节点互指 和 Volume、Vid等信息的更新
	return rack
}
```

