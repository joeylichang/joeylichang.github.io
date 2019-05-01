# bthread概述

bthread是brpc底层的线程库，保证了brpc的高性能，可以说brpc底层的核心。本部分的文章主要是通过代码对bthread进行深入理解，方便brpc的学习。在真正使用brpc时，开发者在很少的场景中会直接使用bthread，这部分工作brpc已经替开发者做好了。对bthread有一个基本的认识（[见M:N线程小节](https://github.com/joeylichang/joeylichang.github.io/blob/master/src/rpc/overview.md)）其实就可以直接学习brpc代码。

## 导航
本系列文章通过以下几部分介绍bthread：

* bthread_init
	* pthread vs. bthread
	* task meta
* bthread_start_urgent/bthread_start_background
	* bthread switch
	* bthread steal
* butex
	* similar to futex
	* bthread and bthread sync
	* bthread and pthread sync
* bthread_join
	* bthread join with butex
* bthread_interrupt
	* bthread wakeup
* bthread_mutex
	* bthread mutex whith butex
