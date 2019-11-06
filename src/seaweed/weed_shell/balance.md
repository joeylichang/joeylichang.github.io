# Balance

Balance 命令的作用是dn间Volume数量均衡。

#### 参数

volume.balance [-c ALL|EACH_COLLECTION|<collection_name>] [-force] [-dataCenter=<data_center_name>]

-c：指定均衡的collect，ALL全部、EACH_COLLECTION按collection逐个执行、collection_name指定collection。

-force：表示执行，默认为false只输出信息不执行。

-dataCenter：指定dc，默认全部执行。

#### 流程

1. 向master拉取全量volume信息。
2. 根据dn.MaxVolumeCount字段对dn进行分类，相同数量的dn之间进行均衡。
3. 根据-c参数调用balanceVolumeServers，按collection执行balance任务。
4. 选取dn上需要搬迁的volume，选取条件collection满足要求，且!v.ReadOnly && v.Size < volumeSizeLimit（可写volume） 或者 v.ReadOnly || v.Size >= volumeSizeLimit（只读volume）
   1. 先balance可写volume，后balance只读volume
5. 对dn进行排序（选出来可以搬迁的volume数量），根据选出来的全部volume个数和dn数量均值作为标准，从排好序的dn收尾节点取volume（优先搬迁容量小的volume），调用LiveMoveVolume（间move介绍）。

注意：整体逻辑相对简单，不赘述源码部分，核心逻辑在LiveMoveVolume，可以参见move介绍。

#### 问题

balance值针对副本不足的情况作了处理，但是对于副本对于的情况没有处理，move的逻辑在copy成功之后（已经mount）但是tail失败之后没有删除目标节点的volume的逻辑可能造成副本对于（可能作者的意思是需要手动调用shell的unmount的逻辑，但是也没有删除文件）。

