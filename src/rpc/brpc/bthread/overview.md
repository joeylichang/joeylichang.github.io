# bthread概述

bthread是brpc底层的线程库，保证了brpc的高性能，可以说brpc底层的核心。本部分的文章主要是通过代码对bthread进行深入理解，方便brpc的学习。在真正使用brpc时，开发者在很少的场景中会直接使用bthread，这部分工作brpc已经替开发者做好了。对bthread有一个基本的认识（[见M:N线程小节](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/overview.md)）其实就可以直接学习brpc代码。

## 导航
本系列文章通过以下几部分介绍bthread：

* [env init](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/init.md)
	* TaskControl / TaskGroup / TaskMeta
	* steal bthread
* [bthread_start_urgent/bthread_start_background](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_start.md)
	* bthread switch
	* bthread start
* [butex](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/butex.md)
	* similar to futex
	* bthread and bthread sync
	* bthread and pthread sync
* [bthread_interrupt](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_interrupt.md)
	* bthread wakeup with butex
* [bthread_join](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_join.md)
	* bthread join with butex
* [bthread_mutex](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/brpc/bthread/bthread_mutex.md)
	* bthread mutex with butex
* bthread_cond
	* bthread cond with bthread_mutex and butex
