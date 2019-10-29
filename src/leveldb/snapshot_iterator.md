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

LevelDB的iterator是全局的迭代器，数据在memtable、immutable、sst文件都有，所以其迭代器是两种（内存是跳跃表、磁盘sst文件）、多个（内存最多两个，磁盘sst文件多个）的聚合。

DBIter：继承自Iterator（纯虚类），对外接口的第一层入口。

Iterator* internal_iter：DBIter的成员变量，组合关系，具体对象为MergingIterator类型。

MergingIterator：继承自Iterator，内部成员变量IteratorWrapper* children_，是一个迭代器数组，包含：MemTableIterator（继承自Iterator，内存跳跃表迭代器）、TwoLevelIterator（继承自Iterator，两层sst文件迭代器，内部包含LevelFileNumIterator，继承自Iterator，一层文件的迭代器）。





