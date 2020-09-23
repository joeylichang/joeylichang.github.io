### 分布式锁

##### 用途

min.io 中所有的存储节点都是对等的架构，不区分主从，min.io 使用分布式锁的方式保证数据的同步，例如：PutObject、GetObject 过程中都使用了分布式锁保证同一时刻只有一个请求可以处理同一个 Object。分布式锁除了在数据的读写之外，在内部一些协调逻辑中也有使用到（后面遇到介绍）。

分布式锁与单机锁相比，需要更多的节点参与进行投票决定请求端是否获锁成功，所以分布式锁可以分位客户端侧和服务端侧。下面也是从这两方面进行介绍的，在介绍之前有必要说明以下几点：

1. min.io 分布式锁的服务端内部维护当前向其请求的锁信息，通过 http 请求完成，底层是 localLocker 数据结构。
2. min.io 分布式锁的客户端内部需要维护所有相关节点的客户端用于加锁解锁等通信使用，如果是远程节点则是正常的远程客户端，如果是本地节点（IP:Port相同，Disk 不同），则直接使用 localLocker 的接口，避免了网络请求。
3. min.io 分布式锁在客户端侧也有相应的抽象，既一个加锁、解锁是否成功有一个统一的接口调用，内部的投票逻辑（多数节点请求成功等）对业务透明，该层抽象是 min.io 提供的分布式锁库dsync.DRWMutex（在 min.io 源码的 /pkg/dsync）。
4. 分布式锁库 dsync.DRWMutex 在初始化阶段，传入的参数包括前两点说明的 远程客户端和 本地客户端，正是通过这些客户端完成分布式通信和相应的投票逻辑。



##### min.io dist-lock-client

min.io 所有的锁只在第一个 zone 中（既，只有第一个 zone 的节点作为锁的服务端），一个zone 中有若干个 set，如果是对一个资源上锁，则对资源进行 hash 取模获取 Set（例如对 Object 名称 hash 取模），如果是对多个资源上锁，则使用固定 Set（资源为空字符串，进行 hash 取模），只有选定的 Set 的内部节点才进行参与相关资源的锁逻辑。

min.io 分布式锁客户端侧在 dsync.DRWMutex 基础上做了一层封装 distLockInstance（本系列文章只关注DistErasure 模式），下面看一下其内部逻辑：

```go
type distLockInstance struct {
	rwMutex *dsync.DRWMutex 		// 分布式存储库
	opsID   string 							// 初始化锁时的随机ID
	ctx     context.Context
}
```

distLockInstance 支持 GetLock、GetRLock、Unlock、RUnlock 四个接口。内部核心逻辑都是直接调用分布式锁库 dsync.DRWMutex 的相应接口（初始化时传入了远程节点的 LockAPI ，在调用上述接口时，会自动向全部节点发送相应的 http 请求以及判断是否全局获锁成功、异常的情况），需要注意的是 dsync.DRWMutex 加锁的参数除了常规的超时等参数，还有 lockSource，表示加锁的源，既 [filename:lineNum:funcName()]，用于标识锁的信息（后面介绍）。



##### min.io dist-lock-server

min.io 分布式锁的服务端主要是针对 localLocker 接口进行维护，其内部有完善的锁信息。结合分布式锁的必要逻辑完成了基本的读写锁接口，除此之外还有一个定期的协程检查所是否过期，避免客户端异常导致资源被长期占用。

###### localLocker

```go
/*
 * 一个 endpoint（Host:Port/DiskPath） 对应一个 分布式锁服务端
 */
type lockRESTServer struct {
	ll *localLocker
}

type localLocker struct {
	mutex    sync.Mutex 	// 加锁、释放锁 修改 map 等资源的局部锁
  endpoint Endpoint 		// 当前分布式服务的 Host:Port/DiskPath
	lockMap  map[string][]lockRequesterInfo 
  // map 的 key 是锁定的资源( bucket/object 等)，可以一次锁定多个资源，如果是多个资源必须保证全部可以上锁才会上锁（写锁的话要看是否全部资源没被抢占）
}

type lockRequesterInfo struct {
	Writer        bool      // 是否是写锁
	UID           string    // 客户端生成的随机 ID，在 distLockInstance 内部生成
	Timestamp     time.Time // 锁初始化的时间
	TimeLastCheck time.Time // 最后一次 check 的时间，分布式锁方式远程节点异常导致无法解锁，需要设置超时时间
	Source        string    // 锁的源，既 distLockInstance 中的 [filename:lineNum:funcName()]
}
```



###### dist-lock-api

1. Lock
   1. 遍历请求的所有资源（bucket/object） , 检查 localLocker.lockMap[resource] 是否所有的资源都没有上锁，否则返回异常
   2. 将所有的请求资源加入 localLocker 的 lockMap 中。
2. RLock
   1. 遍历 lockMap[resource] 对应的数组，记录的是所有的锁信息。
   2. 判断是否有写多，lockRequesterInfo 中  Writer 是否为 true。
   3. 全部资源都没有写锁，则加读锁成功。
3. Unlock
   1. 判断所有资源上的锁是否是写锁，是的话删除相关的 lockRequesterInfo。
4. RUnlock
   1. 查看是否有相应额读锁。
   2. 资源上面没有写锁。
   3. 删除相应的读锁即可。
5. Expired
   1. 查询一个锁是否过期，遍历资源对应的锁，通过 lockRequesterInfo 的 UID 唯一表示一个锁。
6. Health
   1. 只是校验了请求的权限等信息是否正确。



###### dist-lock-background

对于客户端异常情况下，锁可能被长期占有，导致资源不会被释放，常见的解决方案是给分布式锁加一个超时时间一旦超时则节点上的锁资源被释放。min.io 分布式锁有一个协程正式完成相关任务。

1. 分钟级的随机时间，执行如下任务。
2. 获取当前节点上所有过期的锁（超过2min 没有被check）。
3. 向 zone0 中所有的节点发送 Expired 请求，判断节点是否过期。
   1.  对于网络异常、当前节点判断其他节点不再线 等情况都视为成功。
4. 若果判断没有过期的节点个数少于 quorum，则删除 locakname 下对应的 lockinfo。
   1. 读锁 quorum = globalErasureSetDriveCount（EC编码分块数量） / 2
   2. 写锁 quorum = globalErasureSetDriveCount（EC编码分块数量） / 2 + 1







