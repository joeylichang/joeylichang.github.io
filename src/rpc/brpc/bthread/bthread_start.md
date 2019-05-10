# bthread start

## 设计思路
bthread是一个用户态线程库，bthread一定属于某一个pthread，pthread维护一个bthread队列顺序从队列pop出bthread执行。协程与之不同，如果协程内部调用了阻塞的系统调用会切换下其他协程执行（[两者对比见](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/overview.md)），为了能在系统调用完成之后继续执行需要在切换前保存协程的运行时上下文即栈和寄存器信息。根据两者执行的任务模型来看，bthread按顺序执行就好不需要考虑保存运行时上下文，但是bthread库提供了bthread与bthread以及bthread与pthread的同步机制（下篇文章介绍）。如果一个bthread为了等另一个bthread的同步结果而堵塞整个pthread，不将CPU让给后面的bthread执行显然资源是浪费的。为了最大化利用资源bthread需要类似协程的切换机制，即在切换前需要保存栈和寄存器信息。

bthread核心技术之一是steal，即如果有空闲pthread会去steal其他pthread的任务来执行充分的利用CPU资源，pthread维护的队列很容易成为竞争点会成为性能瓶颈（[brpc文档相关介绍](https://github.com/joeylichang/incubator-brpc/blob/master/docs/cn/threading_overview.md#%E5%A4%9A%E6%A0%B8%E6%89%A9%E5%B1%95%E6%80%A7)）。bthread在这里使用了无锁队列[WorkStealingQueue](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/work_stealing_queue.h)。队列并不可能是无限大，在满的时候会强制其他pthread来steal，如果没有空闲pthread会堵塞等待，并发度设置合理发生的概率是很小的，如果所有的CPU资源都被使用率等待也是正常的。

bthread不是一个完备的系统级别的线程，因为它不支持优先级、时间片抢占等，但是bthread库内部对bthread还是有优先级区分的，主要包括一下几个方面：
1. bthread_start_urgent创建的紧急任务需要优先执行，bthread_start_background创建的非紧急任务直接加入队尾等待执行。
2. 任务可以根据用户的设置选择是否通知其他空闲pthread来steal。
3. 非运行bthread的pthread线程创建的任务优先级应该低于运行bthread的pthread任务（例如用户启动了5个线程，其中两个用于网络交互运行bthread，其他线程用于计算将计算结果通知其他两个线程用于网络发送），远程任务通过有锁队列[RemoteTaskQueue](https://github.com/joeylichang/incubator-brpc/blob/master/src/bthread/remote_task_queue.h)维护。

## 源码解析
[接上一遍文章](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/init.md)，TaskGroup创建完毕之后进入TaskGroup::run_main_task开始等待并循环执行TaskMeta（bthread的抽象），其调用路径如下：
```
TaskGroup::run_main_task()
	|_while(wait_task(&tid))
			|_sched_to((TaskGroup** pg, bthread_t next_tid)) 
			--	| ----------------------------------------------------------------------
		递归调用	|	|_sched_to(TaskGroup** pg, TaskMeta* next_meta)                         |
		直到没有	|		|_jump_stack(cur_meta->stack, next_meta->stack)                 |
		任务执行	|		|_task_runner                                                   |
			|			|_ending_sched                                          |
			|				|_sched_to(TaskGroup** pg, TaskMeta* next_meta)	| 
			|--------------------------------------------------------------------------------

```

上述函数的主要作用如下：
1. wait_task(&tid)：初始化之后等待you任务到来唤醒。
2. sched_to(TaskGroup** pg, bthread_t next_tid)：根据tid获取TaskMeta，并且给TaskMeta创建ContextualStack。
3. sched_to(TaskGroup** pg, TaskMeta* next_meta) ：TaskMeta切换。
4. jump_stack：TaskMeta栈切换，切换之后立即执行栈初始化时设置的回调函数task_runner。
5. task_runner：执行用户的回调
6. ending_sched：从队列中弹出任务，根据tid获取TaskMeta并创建ContextualStack，然后调用sched_to(TaskGroup** pg, TaskMeta* next_meta) 。功能等同于wait_task + sched_to(TaskGroup** pg, bthread_t next_tid)。

__*注意: 以下源码有所删减只保留关键逻辑，主要删减了统计信息，这些统计信息是brpc提供强大的内置服务的基础*__

### run_main_task
run_main_task主要代码如下：
```
void TaskGroup::run_main_task() {
    TaskGroup* dummy = this;
    bthread_t tid;
    while (wait_task(&tid)) {
        TaskGroup::sched_to(&dummy, tid);
        DCHECK_EQ(this, dummy);
        DCHECK_EQ(_cur_meta->stack, _main_stack);
        if (_cur_meta->tid != _main_tid) {		#1
            TaskGroup::task_runner(1/*skip remained*/);
        }
    }
}
```
wait_task和sched_to主要功能上面已经描述过了，如果执行到#1表示已经递归将队列里面所有的任务执行完了，此时如果_cur_meta->tid != _main_tid表示当前pthread运行的不是默认bthread即有新的bthread在运行，则继续进入task_runner递归运行。

下面看一下sched_to的主要代码:
```
inline void TaskGroup::sched_to(TaskGroup** pg, bthread_t next_tid) {
    TaskMeta* next_meta = address_meta(next_tid);
    if (next_meta->stack == NULL) {
        ContextualStack* stk = get_stack(next_meta->stack_type(), task_runner);
        if (stk) {
            next_meta->set_stack(stk);
        } else {
            // stack_type is BTHREAD_STACKTYPE_PTHREAD or out of memory,
            // In latter case, attr is forced to be BTHREAD_STACKTYPE_PTHREAD.
            // This basically means that if we can't allocate stack, run
            // the task in pthread directly.
            next_meta->attr.stack_type = BTHREAD_STACKTYPE_PTHREAD;
            next_meta->set_stack((*pg)->_main_stack);
        }
    }
    // Update now_ns only when wait_task did yield.
    sched_to(pg, next_meta);
}


void TaskGroup::sched_to(TaskGroup** pg, TaskMeta* next_meta) {
    TaskGroup* g = *pg;
    // Save errno so that errno is bthread-specific.
    const int saved_errno = errno;

    TaskMeta* const cur_meta = g->_cur_meta;					#1
    // Switch to the task
    if (__builtin_expect(next_meta != cur_meta, 1)) {		
        g->_cur_meta = next_meta;
        // Switch tls_bls
        cur_meta->local_storage = tls_bls;					#2
        tls_bls = next_meta->local_storage;

        if (cur_meta->stack != NULL) {
            if (next_meta->stack != cur_meta->stack) {
                jump_stack(cur_meta->stack, next_meta->stack);			#3
                // probably went to another group, need to assign g again.
                g = tls_task_group;						#4
            }
        }
    }

    while (g->_last_context_remained) {						#5
        RemainedFn fn = g->_last_context_remained;
        g->_last_context_remained = NULL;
        fn(g->_last_context_remained_arg);
        g = tls_task_group;
    }

    // Restore errno
    errno = saved_errno;
    *pg = g;
}
```

void TaskGroup::sched_to(TaskGroup** pg, bthread_t next_tid)的注释解释的已经很详细了，bthread栈的说明见[这里](https://github.com/joeylichang/incubator-brpc/blob/master/docs/cn/memory_management.md#%E6%A0%88)。

void TaskGroup::sched_to(TaskGroup** pg, TaskMeta* next_meta)中有几点说明如下：
1. #1 TaskMeta切换，g->_cur_meta代表当前正在执行的bthread。
2. #2 bthread_local_storage切换，bthread具备内存类似pthread的tls。
3. #3 jump_stack是一段汇编代码，主要作用是栈和寄存器切换，切换后立即调用task_runner（在TaskGroup::sched_to(TaskGroup** pg, bthread_t next_tid) 内get_stack时设置的回调）。
4. #4 重置tls_task_group即当前线程的正在执行的TaskGroup，在使用butex唤醒另外一个任务时可能task_group变化了。
5. #5 _last_context_remained是执行完Task之后的回调，常规操作是清除之前bthread的TaskMeta，如果指定到#5表示当前TaskGroup没有任务执行了，这里是最后一次执行回调。

下面看一下task_runner的主要代码：
```
void TaskGroup::task_runner(intptr_t skip_remained) {
    // NOTE: tls_task_group is volatile since tasks are moved around
    //       different groups.
    TaskGroup* g = tls_task_group;			

    if (!skip_remained) {
        while (g->_last_context_remained) {
            RemainedFn fn = g->_last_context_remained;
            g->_last_context_remained = NULL;
            fn(g->_last_context_remained_arg);
            g = tls_task_group;
        }
    }

    do {
        // Meta and identifier of the task is persistent in this run.
        TaskMeta* const m = g->_cur_meta;

        void* thread_return;
        try {
            thread_return = m->fn(m->arg);
        } catch (ExitException& e) {
            thread_return = e.value();
        } 
        
        // Group is probably changed
        g = tls_task_group;

        // TODO: Save thread_return
        (void)thread_return;

        KeyTable* kt = tls_bls.keytable;
        if (kt != NULL) {
            return_keytable(m->attr.keytable_pool, kt);
            // After deletion: tls may be set during deletion.
            tls_bls.keytable = NULL;
            m->local_storage.keytable = NULL; // optional
        }
        
        // Increase the version and wake up all joiners, if resulting version
        // is 0, change it to 1 to make bthread_t never be 0. Any access
        // or join to the bthread after changing version will be rejected.
        // The spinlock is for visibility of TaskGroup::get_attr.
        {
            BAIDU_SCOPED_LOCK(m->version_lock);				#1
            if (0 == ++*m->version_butex) {
                ++*m->version_butex;
            }
        }
        butex_wake_except(m->version_butex, 0);

        g->_control->_nbthreads << -1;
        g->set_remained(TaskGroup::_release_last_context, m);		#2
        ending_sched(&g);
        
    } while (g->_cur_meta->tid != g->_main_tid);			#3
    
    // Was called from a pthread and we don't have BTHREAD_STACKTYPE_PTHREAD
    // tasks to run, quit for more tasks.
}
```

task_runner中有两处重置了TaskGroup（ g = tls_task_group;），主要原因是task_runner有栈的跳转以及用户回调中可能有切换TaskGroup的可能。
#1的m->version_butex和m->version_butex是利用butex实现的bthread_join，下篇文章重点介绍。
#2_release_last_context回调主要作用是析构当前的TaskMeta，调用处在函数入口的if (!skip_remained)处。#3判断当前的TaskMeta是否是pthread的默认bthread（空闲bthread），如果是的话进入run_main_task的wait_task等待新任务。

最后的ending_sched函数主要功能是wait_task + sched_to比较清晰不在重复。

### bthread_start_urgent
bthread_start_urgent是暴露给用户的接口，bthread库会优先执行bthread_start_urgent创建的任务，底层调用的是TaskGroup::start_foreground，其主要代码如下：
```
int TaskGroup::start_foreground(TaskGroup** pg,
                                bthread_t* __restrict th,
                                const bthread_attr_t* __restrict attr,
                                void * (*fn)(void*),
                                void* __restrict arg) {
    if (__builtin_expect(!fn, 0)) {
        return EINVAL;
    }
    const int64_t start_ns = butil::cpuwide_time_ns();
    const bthread_attr_t using_attr = (attr ? *attr : BTHREAD_ATTR_NORMAL);
    butil::ResourceId<TaskMeta> slot;
    TaskMeta* m = butil::get_resource(&slot);
    if (__builtin_expect(!m, 0)) {
        return ENOMEM;
    }
    CHECK(m->current_waiter.load(butil::memory_order_relaxed) == NULL);
    m->stop = false;
    m->interrupted = false;
    m->about_to_quit = false;
    m->fn = fn;
    m->arg = arg;
    CHECK(m->stack == NULL);
    m->attr = using_attr;
    m->local_storage = LOCAL_STORAGE_INIT;
    m->cpuwide_start_ns = start_ns;
    m->stat = EMPTY_STAT;
    m->tid = make_tid(*m->version_butex, slot);
    *th = m->tid;

    TaskGroup* g = *pg;
    g->_control->_nbthreads << 1;
    if (g->is_current_pthread_task()) {								#1
        // never create foreground task in pthread.
        g->ready_to_run(m->tid, (using_attr.flags & BTHREAD_NOSIGNAL));
    } else {
        // NOSIGNAL affects current task, not the new task.
        RemainedFn fn = NULL;
        if (g->current_task()->about_to_quit) {
            fn = ready_to_run_in_worker_ignoresignal;						#2
        } else {
            fn = ready_to_run_in_worker;							#3
        }
        ReadyToRunArgs args = {
            g->current_tid(),
            (bool)(using_attr.flags & BTHREAD_NOSIGNAL)
        };
        g->set_remained(fn, &args);
        TaskGroup::sched_to(pg, m->tid);
    }
    return 0;
}
```

函数入口是TaskMeta在之前的文章介绍过不在赘述，这里重点说明以下几点：
1. #1表示当前pthread空闲，ready_to_run将任务加入WorkStealingQueue队列，根据用户传入的bthread_attr决定是否通知pthread来steal。
2. #2 g->current_task()->about_to_quit表示正要执行的队列头的TaskMeta被用户设置了about_to_quit属性表示可以delay执行，在当前的urgent任务执行之前调用set_remained的回调函数将其加入队列尾并且不会通知空闲pthread来steal。
3. #3在当前的urgent任务执行之前调用set_remained的回调函数将队列头的任务加入队列尾并且通知空闲pthread来steal。

### bthread_start_background
bthread_start_background同样是暴露给用户的接口，bthread库只是将bthread_start_background创建的任务加入队列尾，底层调用的是TaskGroup::start_background，其主要代码如下：
```
template <bool REMOTE>
int TaskGroup::start_background(bthread_t* __restrict th,
                                const bthread_attr_t* __restrict attr,
                                void * (*fn)(void*),
                                void* __restrict arg) {
    if (__builtin_expect(!fn, 0)) {
        return EINVAL;
    }
    const int64_t start_ns = butil::cpuwide_time_ns();
    const bthread_attr_t using_attr = (attr ? *attr : BTHREAD_ATTR_NORMAL);
    butil::ResourceId<TaskMeta> slot;
    TaskMeta* m = butil::get_resource(&slot);
    if (__builtin_expect(!m, 0)) {
        return ENOMEM;
    }
    CHECK(m->current_waiter.load(butil::memory_order_relaxed) == NULL);
    m->stop = false;
    m->interrupted = false;
    m->about_to_quit = false;
    m->fn = fn;
    m->arg = arg;
    CHECK(m->stack == NULL);
    m->attr = using_attr;
    m->local_storage = LOCAL_STORAGE_INIT;
    m->cpuwide_start_ns = start_ns;
    m->stat = EMPTY_STAT;
    m->tid = make_tid(*m->version_butex, slot);
    *th = m->tid;
    if (using_attr.flags & BTHREAD_LOG_START_AND_FINISH) {
        LOG(INFO) << "Started bthread " << m->tid;
    }
    _control->_nbthreads << 1;
    if (REMOTE) {
        ready_to_run_remote(m->tid, (using_attr.flags & BTHREAD_NOSIGNAL));		#1
    } else {
        ready_to_run(m->tid, (using_attr.flags & BTHREAD_NOSIGNAL));			#2
    }
    return 0;
}
```
函数入口的初始化代码不再重复，做一下几点说明：
1. #1将任务加入到RemoteTaskQueue队列。
2. #2将任务加入WorkStealingQueue队列尾。
3. REMOTE == true，调用bthread_start_background或者bthread_start_urgent的pthread非运行bthread的pthread线程（前文bthread优先级的第三种情况）。
