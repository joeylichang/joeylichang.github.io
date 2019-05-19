# bthread interrupt

## 设计思路
bthread_interrupt提供了一种bthread之间可靠的唤醒机制，常见的使用方式如下：

|                                pthread A|                                                 bthread B|
|---------------------------------------|-------------------------------------------------------|
|                    set stopping flag |                                                                 |
| bthread_interrupt(bthread B) |                                                                 |
|                                               |                                                   wake up |
|                                               |                                see the flag and quit |
|                                               | may block again if the flag is unchanged |

需要注意的是如果bthread正堵塞bthread_wait中，bthread_interrupt由于是可靠的唤醒机制，即不会丢失，所以仍能唤醒堵塞在bthread_wait的bthread，但是bthread_wait应该非正确值告知bthread_wait调用者有bthread_interrupt已经唤醒了，后续是否继续bthread_wait由调用者决定。

## 源码解析

```
int TaskGroup::interrupt(bthread_t tid, TaskControl* c) {
    // Consume current_waiter in the TaskMeta, wake it up then set it back.
    ButexWaiter* w = NULL;
    uint64_t sleep_id = 0;
    int rc = interrupt_and_consume_waiters(tid, &w, &sleep_id);		#1
    if (rc) {
        return rc;
    }
    // a bthread cannot wait on a butex and be sleepy at the same time.
    CHECK(!sleep_id || !w);
    if (w != NULL) {
        erase_from_butex_because_of_interruption(w);			#2
        // If butex_wait() already wakes up before we set current_waiter back,
        // the function will spin until current_waiter becomes non-NULL.
        rc = set_butex_waiter(tid, w);					#3
        if (rc) {
            LOG(FATAL) << "butex_wait should spin until setting back waiter";
            return rc;
        }
    } 
    return 0;
}
```

1. #1获取当前bthread的ButexWaiter（ButexWaiter介绍见[bthread](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md)）。
2. #2替换当前bthread为NULL，防止butex_wait唤醒ButexWaiter，同时唤醒bthread。
3. #3在#2唤醒bthrad之后，用之前的ButexWaiter加回原来的bthread中，当有butex_wake唤醒butex_wait时可以唤醒它，但是此时butex_wait会返回-1表示有期间有其他bthread调用bthread_interrupt唤醒过。
4. [bthread](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md)中butex_wait在唤醒之后有如下代码：

```
// If current_waiter is NULL, TaskGroup::interrupt() is running and using bbw.
// Spin until current_waiter != NULL.
BT_LOOP_WHEN(bbw.task_meta->current_waiter.exchange(
                     NULL, butil::memory_order_acquire) == NULL,
                 30/*nops before sched_yield*/);
```

表示bthread_interrupt已经用NULL替换了ButexWaiter，需要butex_wait无锁重试若干次等待bthread_interrupt执行完毕。butex_wait返回逻辑如下：

```

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
```

当前bthread被interrupt之后会设置task_meta->interrupted标志，并返回-1，设置errno = EINTR。
