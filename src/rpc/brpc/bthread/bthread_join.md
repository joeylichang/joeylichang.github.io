# bthread_join

## 设计思路
bthread_join提供了等待某一个bthread结束的机制，功能类似pthread_join。bthread_join底层同步使用的是[butex](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md)，每个bthread执行完毕后会调用pthread_wake唤醒调用bthread_join等待当先bthread的bthread。如果调用bthread_join之前关注的bthread已经运行结束了，那么应该立即返回，由于bthread的内存是重复利用使用版本号进行区分是否过期，所以在bthread_join和pthread_wake都需要通过版本号进行判断等待的bthread是否过期。

## 源码解析
```
int TaskGroup::join(bthread_t tid, void** return_value) {
    if (__builtin_expect(!tid, 0)) {  // tid of bthread is never 0.
        return EINVAL;
    }
    TaskMeta* m = address_meta(tid);
    if (__builtin_expect(!m, 0)) {
        // The bthread is not created yet, this join is definitely wrong.
        return EINVAL;
    }
    TaskGroup* g = tls_task_group;
    if (g != NULL && g->current_tid() == tid) {
        // joining self causes indefinite waiting.
        return EINVAL;
    }
    const uint32_t expected_version = get_version(tid);		
    while (*m->version_butex == expected_version) {				#1
        if (butex_wait(m->version_butex, expected_version, NULL) < 0 &&		#2
            errno != EWOULDBLOCK && errno != EINTR) {
            return errno;
        }
    }
    if (return_value) {
        *return_value = NULL;
    }
    return 0;
}
```

1. #1获取当前bthread的本版号，关于bthread的版本号间[brpc的bthread_t介绍](https://github.com/joeylichang/incubator-brpc/blob/master/docs/cn/memory_management.md#%E7%94%9F%E6%88%90bthread_t)。
2. #2[butex_wait](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md)等待关注的bthread结束之后唤醒当前join的bthread。


关注的bthread执行完会后唤醒调用bthread_join的bthread：
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
        butex_wake_except(m->version_butex, 0);				#2

        g->_control->_nbthreads << -1;
        g->set_remained(TaskGroup::_release_last_context, m);	
        ending_sched(&g);
        
    } while (g->_cur_meta->tid != g->_main_tid);
    
    // Was called from a pthread and we don't have BTHREAD_STACKTYPE_PTHREAD
    // task
```

1. task_runner主要是处理了，调用用户创建bthread时传入的回调函数、bls、回调等逻辑，详情见[bthread start](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_start.md)。
2. #1提升bthread的版本号，表示当先bthread执行完毕，当前bthread的资源释放给内存池可以循环利用。
3. #2唤醒调用bthread_join的bthread。