# bthread_mutex

## 设计思路
bthread_mutex提供bthread与bthread、bthread与pthread之间的锁机制，本部分主要介绍[futex](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md#futex)中描述的用户态部分与之前介绍的butex_wait、butex_wake组成完整的一套类futex（butex_wait、butex_wake也是用户态，futex_wait、futex_wake是内核态）的机制。

bthread_mutex在实现上与前述文章中介绍的[futex](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md#futex)稍有不同，因为bthread_mutex每次只需要唤醒一个等待的bthread即可所以butex维护的变量不用出现负数，但是要保证butex变量读取变更的原子性为此bthread_mutex对butex变量进行了重新解释增加了lock、content两个变量（稍后介绍）。

## 源码解析

### bthread_mutex_init
```
int bthread_mutex_init(bthread_mutex_t* __restrict m,
                       const bthread_mutexattr_t* __restrict) {
    bthread::make_contention_site_invalid(&m->csite);
    m->butex = bthread::butex_create_checked<unsigned>();	#1
    if (!m->butex) {
        return ENOMEM;
    }
    *m->butex = 0;				
    return 0;
}

typedef struct {
    unsigned* butex;
    bthread_contention_site_t csite;		// 统计信息，暂时忽略
} bthread_mutex_t;

struct BAIDU_CACHELINE_ALIGNMENT Butex {
    Butex() {}
    ~Butex() {}

    butil::atomic<int> value;
    ButexWaiterList waiters;					#2
    internal::FastPthreadMutex waiter_lock;
};

struct MutexInternal {						#3
    butil::static_atomic<unsigned char> locked;
    butil::static_atomic<unsigned char> contended;
    unsigned short padding;					// 预留占位
};
```

1. #1 m->bute是从内存资源池获取的Butex架构的value变量，即前文介绍的butex变量。
2. #2 是记录butex的bthread等待队列，每次从其中获取bthread进行唤醒。
3. #3 MutexInternal是bthread_mutex内部对butex变量，即#1处的m->butex 重新解释（类型转换），MutexInternal 的使用后面介绍。

### bthread_mutex_lock
```
int bthread_mutex_lock(bthread_mutex_t* m) {
    bthread::MutexInternal* split = (bthread::MutexInternal*)m->butex;		#0
    if (!split->locked.exchange(1, butil::memory_order_acquire)) {		#1
        return 0;
    }
    // Don't sample when contention profiler is off.
    if (!bthread::g_cp) {
        return bthread::mutex_lock_contended(m);
    }
}

const MutexInternal MUTEX_CONTENDED_RAW = {{1},{1},0};
const MutexInternal MUTEX_LOCKED_RAW = {{1},{0},0};
#define BTHREAD_MUTEX_CONTENDED (*(const unsigned*)&bthread::MUTEX_CONTENDED_RAW)
#define BTHREAD_MUTEX_LOCKED (*(const unsigned*)&bthread::MUTEX_LOCKED_RAW)

inline int mutex_lock_contended(bthread_mutex_t* m) {
    butil::atomic<unsigned>* whole = (butil::atomic<unsigned>*)m->butex;
    while (whole->exchange(BTHREAD_MUTEX_CONTENDED) & BTHREAD_MUTEX_LOCKED) {		#2
        if (bthread::butex_wait(whole, BTHREAD_MUTEX_CONTENDED, NULL) < 0 &&		#3
            errno != EWOULDBLOCK && errno != EINTR/*note*/) {
            // a mutex lock should ignore interrruptions in general since
            // user code is unlikely to check the return value.
            return errno;
        }
    }
    return 0;
}
```

1. #1用1替换MutexInternal的locked变量，如果之前的值是0，表示当前锁没有被占用直接返回。
2. 否则当前的锁有人被占用，#2用{{1},{1},0}替换之前值如果之前的lock变量仍未1，表示#1与#2之间的时间窗口内锁没有释放或者释放了又被其他bthred占用了（lock变量的作用就是针对时间窗口的）。
3. #3调用butex_wait等待锁的释放。

### bthread_mutex_unlock
```
int bthread_mutex_unlock(bthread_mutex_t* m) {
    butil::atomic<unsigned>* whole = (butil::atomic<unsigned>*)m->butex;

    const unsigned prev = whole->exchange(0, butil::memory_order_release);
    // CAUTION: the mutex may be destroyed, check comments before butex_create
    if (prev == BTHREAD_MUTEX_LOCKED) {			#1
        return 0;
    }
    // Wakeup one waiter
    if (!bthread::is_contention_site_valid(saved_csite)) {
        bthread::butex_wake(whole);			#2
        return 0;
    }
}
```

1. #1表示lock变量为1，content为0，表示没有其他bthread在等待锁，直接返回即可。
2. #2唤醒一个等待锁的bthread。