# snapshot

LevelDB的snapshot的本质是一个seq，例如GetSnapshot 既当前binlog的seq记录下来，后续的Get、Itreator在option如果指定了snapshot，后续操作会key时的seq会与snapshot进行比较。compact会考虑最久远的snapshot，不会压缩其之后的数据。

源码如下：

```c++
const Snapshot* DBImpl::GetSnapshot() {
  MutexLock l(&mutex_);
  // snapshots_ 是一个链表
  return snapshots_.New(versions_->LastSequence());
}

void DBImpl::ReleaseSnapshot(const Snapshot* s) {
  MutexLock l(&mutex_);
  snapshots_.Delete(reinterpret_cast<const SnapshotImpl*>(s));
}
```



# Iterator

