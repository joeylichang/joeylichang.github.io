# WDR

###  Write && Delete

seaweed的写和删除的逻辑基本一致，主要包括一下步骤：

1. 调用assign接口向master申请fid（volume不够会alloc，见assign逻辑）。
2. assign返回一个dn的addr，向该addr发起write/delete请求。
3. 当前dn写入成功之后，向master发送一个查询请求（vi其他的dn），当前dn向其他dn并发发起写请求。
4. 所有dn都写入成功，则写入成功，否则失败。

代码逻辑比较清晰简单不在此赘述。

### Read

1. 向master发起LookupVolume请求，返回vid所在的所有的dn。
2. 向某一个dn发起Query请求（支持多核fid查询），返回用户数据（有一些参数可设置）。