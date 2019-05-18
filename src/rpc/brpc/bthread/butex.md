# butex

## futex
butex是bthread库提供的一种同步机制，目的是完成bthread与bthread、bthread与pthread之间的同步，它也是bthread_mutex、bthread_cond的底层核心。butex的实现模仿了futex，futex分为用户态同步和系统调用，butex除了需要和pthread之间进行同步之外全部是用户态操作。有必要在介绍butex之前先了解一下futex，[linux futex浅析](https://yq.aliyun.com/articles/6043)一文中关于futex介绍的比较详细且通俗易懂，在这里不再重复，关于futex总结如下几方面：

##### 背景 && 目标
进程或者线程的同步都需要系统调用完成，然而临界区（比如：锁、条件变量等同于方式的临界区）在很多时刻并没有产生竞争（没有等待现象），所以造成一定的资源浪费，需要一种机制能够保证在必要时才进行系统调用，即先在用户态做判断如果没有竞争线程继续执行，如果有竞争再调用系统调用进行等待，从而较少不必要的系统调用。futex的设计目标正是解决上述问题。

##### 实现
正如前所述futex需要具备两方面内容：
* 用户态同步机制
	* 用户抢占同步资源时，会先对futex变量（进程、线程共享的一段内存中）进行“down”操作，对变量减1。
		* 如果变量变为0，表示没有竞争，线程照常运行。
		* 如果变量变为负数，标志有竞争，则进程需要挂起等待其他持有者释放资源之后唤醒。
	* 用户释放同步资源之后，会对futex变量进行“up”操作，对变量加1.
		* 如果变量是0，继续加1（1表示有资源），表示没有竞争（即没有线程在等待）当前线程照常运行。
		* 如果变量是负数，表示有竞争（即有其他线程在等待）当前线程需要唤醒若干（1个、多个或者所有）等待的线程去抢占资源。
* 内核态睡眠唤醒机制
	* 根据前面介绍可知，内核态的睡眠和唤醒机制应该是针对锁粒度的，即根据所进行唤醒和睡眠（这里的锁就是futex的变量），所以需要为每个futex维护一个线程队列。
	* futex的睡眠和唤醒的系统调用是futex_wait、futex_wake，原型分别是int futex_wait(int *uaddr, int val); 和 int futex_wake(int *uaddr, int n);
		* uaddr是futex的指针，不同进程的指针值是不一样的。
		* val是当前线程调用futex_wait获取的futex变量值，作用类似cas，因为多进程或者多线程编程中有时间片抢占问题，即调用futex_wait、futex_wake时和之前的判断值（1、0或者负数）之间有时间窗口值可能变化。如果不是之前的判断值val，需要调用处重试。

## butex设计原理
本部分介绍的butex源码主要是针对上述futex的内核部分的设计（在butex中实现在了用户态，因为butex是用户态线程库），至于上述futex的用户态逻辑在后续的bthread_mutex中介绍。

上述的futex变量在butex对应架构体Butex，Butex中除了计数器外还包括ButexWaiterList记录这个等待Butex同步变量的bthread，一个操作者这个list的锁。

## 源码解析
由于篇幅有限本部分重点关注bthread与bthread的同步，bthread与pthread的同步逻辑大致相同不再重复，超时逻辑相对对立也是本分部介绍的重点。

|bthread A of pthrad A|bthread A of pthred A/B/Other |                bthread B of pthreadB|
|----------------------------|----------------------------------------|-------------------------------------------|
|butex_wait                |                                                |                                                    |
|TaskGroup::sched    |                                                |                                                    |
|_wait_for_butex        |                                                |                                                    |
|TaskGroup::sched    |                                                |                                                    |
|{run other task}         |                                                |                                                    |
|                                 |                                                |                                 butex_wake|
|                                 |                                                | ready_to_run_remote/exchange|
|                                 |                    return butex_wait|                                                    |

butex_wait 和  butex_wake调用关系如下，下面看一下源码。


### butex_wait 

```
int butex_wait(void* arg, int expected_value, const timespec* abstime) {
    Butex* b = container_of(static_cast<butil::atomic<int>*>(arg), Butex, value);
    if (b->value.load(butil::memory_order_relaxed) != expected_value) {
        errno = EWOULDBLOCK;
        // Sometimes we may take actions immediately after unmatched butex,
        // this fence makes sure that we see changes before changing butex.
        butil::atomic_thread_fence(butil::memory_order_acquire);
        return -1;
    }
    TaskGroup* g = tls_task_group;
    if (NULL == g || g->is_current_pthread_task()) {
        return butex_wait_from_pthread(g, b, expected_value, abstime);
    }
    ButexBthreadWaiter bbw;
    // tid is 0 iff the thread is non-bthread
    bbw.tid = g->current_tid();
    bbw.container.store(NULL, butil::memory_order_relaxed);
    bbw.task_meta = g->current_task();
    bbw.sleep_id = 0;
    bbw.waiter_state = WAITER_STATE_READY;
    bbw.expected_value = expected_value;
    bbw.initial_butex = b;
    bbw.control = g->control();
 
    // release fence matches with acquire fence in interrupt_and_consume_waiters
    // in task_group.cpp to guarantee visibility of `interrupted'.
    bbw.task_meta->current_waiter.store(&bbw, butil::memory_order_release);
    g->set_remained(wait_for_butex, &bbw); 	#1
    TaskGroup::sched(&g);			#2

    // If current_waiter is NULL, TaskGroup::interrupt() is running and using bbw.
    // Spin until current_waiter != NULL.
    BT_LOOP_WHEN(bbw.task_meta->current_waiter.exchange(
                     NULL, butil::memory_order_acquire) == NULL,
                 30/*nops before sched_yield*/);	#3

    bool is_interrupted = false;
    if (bbw.task_meta->interrupted) {
        // Race with set and may consume multiple interruptions, which are OK.
        bbw.task_meta->interrupted = false;
        is_interrupted = true;
    }
    // If timed out as well as value unmatched, return ETIMEDOUT.
    if (WAITER_STATE_TIMEDOUT == bbw.waiter_state) {
        errno = ETIMEDOUT;
        return -1;
    } else if (WAITER_STATE_UNMATCHEDVALUE == bbw.waiter_state) {
        errno = EWOULDBLOCK;
        return -1;
    } else if (is_interrupted) {
        errno = EINTR;
        return -1;
    }
    return 0;
}
```
上述源码有三处需要注意：
1. #2TaskGroup::sched在[bthread start](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_start.md)中介绍过将会跳过当前bthread，进行后续的循环从队列中pop出bthread去执行，此时调用butex_wait的bthread的上下文（栈、寄存器）会保存在TaskMeta的Stack中，所以在#2当前的bthread会被堵塞。
2. #1是[bthread start](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_start.md)中介绍的回调函数，在进行下个bthread之前会调用这个回调，wait_for_butex的作用是将ButexBthreadWaiter加入Butex的等待对联中。
3. #3在后面的bthread_interpert（bthread之间的唤醒机制）中介绍。

### butex_wake
```
int butex_wake(void* arg) {
    Butex* b = container_of(static_cast<butil::atomic<int>*>(arg), Butex, value);
    ButexWaiter* front = NULL;
    {
        BAIDU_SCOPED_LOCK(b->waiter_lock);
        if (b->waiters.empty()) {
            return 0;
        }
        front = b->waiters.head()->value();
        front->RemoveFromList();
        front->container.store(NULL, butil::memory_order_relaxed);
    }
    if (front->tid == 0) {
        wakeup_pthread(static_cast<ButexPthreadWaiter*>(front));
        return 1;
    }
    ButexBthreadWaiter* bbw = static_cast<ButexBthreadWaiter*>(front);
    unsleep_if_necessary(bbw, get_global_timer_thread());
    TaskGroup* g = tls_task_group;
    if (g) {
        TaskGroup::exchange(&g, bbw->tid);
    } else {
        bbw->control->choose_one_group()->ready_to_run_remote(bbw->tid);
    }
    return 1;
}
```

源码主要逻辑比较清晰从Butex的瞪大队列中去除第一个ButexWaiter，获取tid用于调度。需要注意的是这里调用tid的group（pthread）肯可能发生了变化，并不一定是调用butex_wait的group，choose_one_group或者exchange都会调用sched_to，在其中会调用jump_stack（[bthread start](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_start.md)中有介绍）在stack切换之后会切换栈和寄存器信息，从butex_wait的断点（TaskGroup::sched）之后开始继续执行。
