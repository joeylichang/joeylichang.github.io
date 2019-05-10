# env init

## 设计思路 
bthread是M:N线程库，即bthrad : pthread 为 M : N（M >= N）。bthread的核心技术之一是steal bthread，即当前pthead无bthread执行时，会去其他pthread的队列中steal bthread执行，充分的利用了多核的能力。

### 重要数据结构
* [TaskControl](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/task_control.h)：负责管理所有的pthread。
* [TaskGroup](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/task_group.h)：对应一个具体的pthread。
* [TaskMeta](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/task_meta.h)：bthread的元信息。
* [ParkingLot](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/parking_lot.h)
	* TaskGroup之间的条件变量机制。
	* 为什么不用pthread_cond，系统调用会增加开销。
	* 在哪些情况下使用：
		1. 如果一个紧急的bthread被创建了，恰好当前pthread的bthread队列有任务且其他pthread空闲，通知其他pthread来steal bthread。
		2. 防止遗漏steal bthread，通知空闲pthread来steal bthread（下文详细介绍）。
* [bthread_t](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/types.h)
	* bthread的标识，uint64_t类型。
	* version（32bit） + offset（32bit）这种设计主要是考虑性能和ABA问题，详细设计见[资源管理](https://github.com/joeylichang/incubator-brpc/blob/master/docs/cn/memory_management.md#%E7%94%9F%E6%88%90bthread_t)。

***注意：为避免冗长的代码影响主要思想的表述，以下源码均有删减***

## 源码解析
bthread的创建函数（bthread_start_urgent/bthread_start_background）如果发现还没有创建TaskControl会调用get_or_new_task_control创建TaskControl。  
get_or_new_task_control内部调用链如下：

```
TaskControl::init                                          //根据并发度调用N次pthread_create。
	|_pthread_create(入口函数worker_thread)
		|_TaskControl::create_group
		|	|_TaskGroup::init                  //初始化默认bthread
		|	|_TaskControl::_add_group          //添加group到control
		|		|_TaskControl::signal_task //通知其他group
		|_TaskGroup::run_main_task                 //调用用户创建bthread的回调函数
			|_TaskGroup::wait_task             //没有bthread执行时，等待任务通知
		
```

TaskControl全局唯一，在初始化阶段根据并发度的配置创建N个pthread用于运行bthread。pthread入口函数worker_thread负责TaskGroup（pthread一一对应）初始化、执行bthread、销毁等操作。

整个初始化的流程代码比较清晰简洁，本文不做过多细枝末节的源码讲解，重点关注add_group、destroy_group、steal_task、signal_task / wait_task这几部分的设计和处理。
重点关注TaskControl内部关于TaskGroup管理的两个成员变量：_ngroup（计数）、_groups（指针数组）。为了提升steal task效率做了一下几点处理：
1. _ngroup的同步没有使用锁，使用了原子操作，某一个空闲的group在steal task时根据_ngroup遍历_groups寻找目标bthread。
2. 在_destory_group时，没有直接删除_groups中的数据，而是用最后一个group指针覆盖需要删除的goup指针。
3. 在add_group时，调用signal_task防止因覆盖最后一个group（add_group直接添加到_groups的最后）而漏掉最后一个group上的bthread未被steal task看到。

下面结合源码具体看一下：

### add_group

```
int TaskControl::_add_group(TaskGroup* g) {
    if (__builtin_expect(NULL == g, 0)) {
        return -1;
    }
    std::unique_lock<butil::Mutex> mu(_modify_group_mutex);
    if (_stop) {
        return -1;
    }
    size_t ngroup = _ngroup.load(butil::memory_order_relaxed);
    if (ngroup < (size_t)BTHREAD_MAX_CONCURRENCY) {
        _groups[ngroup] = g;
        _ngroup.store(ngroup + 1, butil::memory_order_release);
    }
    mu.unlock();
    // See the comments in _destroy_group
    // TODO: Not needed anymore since non-worker pthread cannot have TaskGroup
    signal_task(65536);
    return 0;
}
```

1. _modify_group_mutex的锁是用来修改_groups，add_group、destroy_group操作并不频繁使用锁并无大碍，steal task主要关注_ngroup变量，使用的是原子操作。
2. 源码最后调用了signal_task通知其他空闲group 去 steal bthread，这要原因如下：
	1. 场景：
		1. 设想如下场景，ngroup = 5，_groups存储了5个group指针，现在需要删除索引为2的的group，destroy_group的做法是用最后一个goup指针即索引为4的group覆盖索引为2的group。
		2. 此时有另外一个group假设是索引为0的group正在steal task，如果它在steal之前获取的ngroup = 5而不是4（没有用锁而用原子操作是完全有可能的），开始遍历_groups进行steal，此时索引为2、5的两个group是一个指针。
		3. 恰好它遍历过了索引为2的group，正常情况继续遍历还是会遍历索引为5的group去steal，即及时错过了索引2但是没有错过索引5（他俩是一个group），最终不会漏steal。
	2. 问题
		1. 如果steal task在遍历到索引为5的group之前，调用了_add_group会覆盖索引为5的group，这样索引5与索引2不再是一个group，那么原本最后一个group因为跑到了索引2，而被了。
	3. 解决办法
		1. 在_add_group之后signal_task通知空闲的group重新遍历_groups来steal task避免上述的遗漏问题。
	4. 上述处理逻辑的复杂度原因来自于，用_ngroup的原子操作替代锁来换去steal task的性能，这也正是无锁编程的难点之一。


### destroy_group

```
int TaskControl::_destroy_group(TaskGroup* g) {
    if (NULL == g) {
        LOG(ERROR) << "Param[g] is NULL";
        return -1;
    }
    if (g->_control != this) {
        LOG(ERROR) << "TaskGroup=" << g
                   << " does not belong to this TaskControl=" << this;
        return -1;
    }
    bool erased = false;
    {
        BAIDU_SCOPED_LOCK(_modify_group_mutex);
        const size_t ngroup = _ngroup.load(butil::memory_order_relaxed);
        for (size_t i = 0; i < ngroup; ++i) {
            if (_groups[i] == g) {
                // No need for atomic_thread_fence because lock did it.
                _groups[i] = _groups[ngroup - 1];
                // Change _ngroup and keep _groups unchanged at last so that:
                //  - If steal_task sees the newest _ngroup, it would not touch
                //    _groups[ngroup -1]
                //  - If steal_task sees old _ngroup and is still iterating on
                //    _groups, it would not miss _groups[ngroup - 1] which was 
                //    swapped to _groups[i]. Although adding new group would
                //    overwrite it, since we do signal_task in _add_group(),
                //    we think the pending tasks of _groups[ngroup - 1] would
                //    not miss.
                _ngroup.store(ngroup - 1, butil::memory_order_release);
                //_groups[ngroup - 1] = NULL;
                erased = true;
                break;
            }
        }
    }

    // Can't delete g immediately because for performance consideration,
    // we don't lock _modify_group_mutex in steal_task which may
    // access the removed group concurrently. We use simple strategy here:
    // Schedule a function which deletes the TaskGroup after
    // FLAGS_task_group_delete_delay seconds
    if (erased) {
        get_global_timer_thread()->schedule(
            delete_task_group, g,
            butil::microseconds_from_now(FLAGS_task_group_delete_delay * 1000000L));
    }
    return 0;
}
```

结合注释 和 在 add_group中的介绍，对于destroy_group的逻辑应该不难理解，需要注意一下节点。
1. destroy_group使用了覆盖删除的方式，难点在于_ngroup原子变量的可见性，注释的解释已经很清晰了不在重复，其中比较难理解的是为什么在_add_group中调用signal_task，前面已经描述了。
2. destroy_group处理覆盖删除，还是用了延时删除，主要原因是steal task仍然可能会遍历最后一个group，如果直接删除会core，根本原因还是因为使用原子变量代替锁。
3. _ngroup有的地方用memory_order_relaxed内存序，有的时候用memory_order_release内存序：
	1. memory_order_relaxed 性能优于 memory_order_release，在一些可容忍_ngroup不是严格准确（一些读的场景）或者有其他同步机制（比如锁）的情况下可以用memory_order_relaxed。
	2. 在_destroy_group、_add_group中对于_ngroup的load使用memory_order_relaxed，因为在_modify_group_mutex内部完成的即已经存在了同步机制，不需要性能更差的memory_order_release。
	3. 在_destroy_group、_add_group中对于_ngroup的store都使用了memory_order_release，需要向所有的线程同步这个结果，比如steal task需要看到这个结果。

### steal_task
```
bool TaskControl::steal_task(bthread_t* tid, size_t* seed, size_t offset) {
    // 1: Acquiring fence is paired with releasing fence in _add_group to
    // avoid accessing uninitialized slot of _groups.
    const size_t ngroup = _ngroup.load(butil::memory_order_acquire/*1*/);
    if (0 == ngroup) {
        return false;
    }

    // NOTE: Don't return inside `for' iteration since we need to update |seed|
    bool stolen = false;
    size_t s = *seed;
    for (size_t i = 0; i < ngroup; ++i, s += offset) {
        TaskGroup* g = _groups[s % ngroup];
        // g is possibly NULL because of concurrent _destroy_group
        if (g) {
            if (g->_rq.steal(tid)) {
                stolen = true;
                break;
            }
            if (g->_remote_rq.pop(tid)) {
                stolen = true;
                break;
            }
        }
    }
    *seed = s;
    return stolen;
}
```

1. _rq 和 _remote_rq 是两个队列每个group个一对，下篇文章介绍。
2. 源码开头 获取_ngroup数值用的是memory_order_acquire内存序，严格的保证获取数值的正确性，保证是最后一次调用add_group、destroy_group之后修改的值，与其中的memory_order_release对应，方式有group漏steal。

### group_init
TaskControl的create_group内部调用了TaskGroup::init初始化group，它也是创建group的唯一入口，下面看一下其源码：
```
int TaskGroup::init(size_t runqueue_capacity) {
    if (_rq.init(runqueue_capacity) != 0) {			// bthread队列
        LOG(FATAL) << "Fail to init _rq";						
        return -1;
    }
    if (_remote_rq.init(runqueue_capacity / 2) != 0) {		// bthread远程队列后续blog介绍
        LOG(FATAL) << "Fail to init _remote_rq";
        return -1;
    }
    ContextualStack* stk = get_stack(STACK_TYPE_MAIN, NULL);	// bthread运行环境（栈和寄存器信息）
    if (NULL == stk) {
        LOG(FATAL) << "Fail to get main stack container";
        return -1;
    }
    butil::ResourceId<TaskMeta> slot;			// TaskMeta资源的资源ID
    TaskMeta* m = butil::get_resource<TaskMeta>(&slot); // 申请TaskMeta内存并返回资源ID，内存重用
    if (NULL == m) {
        LOG(FATAL) << "Fail to get TaskMeta";
        return -1;
    }
    m->stop = false;					// 是否停止的标志
    m->interrupted = false;				// 是否被其他bthread唤醒过，bthread_interrupt函数会操作该标志
    m->about_to_quit = false;				// 是否可以delay被调度
    m->fn = NULL;					// fn是创建bthread时用户传入的入口函数，arg是fn参数
    m->arg = NULL;
    m->local_storage = LOCAL_STORAGE_INIT;		// bthread的tls
    m->cpuwide_start_ns = butil::cpuwide_time_ns();	// 统计信息
    m->stat = EMPTY_STAT;			
    m->attr = BTHREAD_ATTR_TASKGROUP;			// bthread的attr，用户可指定
    m->tid = make_tid(*m->version_butex, slot);		// tid，bthread的唯一标识，m->version_butex 是bthread的同步机制，在这里主要用于bthread的join
    m->set_stack(stk);					// bthread运行环境（栈和寄存器信息）

    _cur_meta = m;					// _cur_meta当前group正在运行的bthread
    _main_tid = m->tid;					// group默认的bthread，group空闲的时候运行的bthread，通过_cur_meta与_main_tid的比较判断group是否有bthread在运行
    _main_stack = stk;
    _last_run_ns = butil::cpuwide_time_ns();
    return 0;
}
```

1. TaskMeta是bthread的抽象，是非常重要的数据结构，成员变量的含义在注释中进行了说明，后面会逐步详细介绍。
2. TaskMeta、tid等很多数据结构在内存上都是用了version + offset的方式实现内存重用，详细见[brpc文档](https://github.com/joeylichang/incubator-brpc/blob/master/docs/cn/memory_management.md)。

### signal_task / wait_task
1. signal_task / wait_task内部使用了ParkingLot完成了group之间的通信（ParkingLot实现了条件变量机制）。
2. ParkingLot底层使用bthread实现的用户态futex（futex_wake_private、futex_wait_private），详情见[ParkingLot源码](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/parking_lot.h)。
