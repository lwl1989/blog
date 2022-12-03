CreateTime:2021-09-22 15:46:21.0

# io模型

计算机的io模型区分为多种，目前用的最多的也就是nio、epoll、select。

结合不同场景使用不同的io模型才是正解。

具体可以查看我之前写的io模型演进。[io模型演进](https://my.oschina.net/lwl1989/blog/5006295 "io模型演进")

# golang中网络io

golang天然适合并发，为什么？一个是轻量级的协程，二个是将复杂的io进行了抽象化，简化了流程。

比如我们简单的访问一个http服务，几行简单的代码就能实现:

```
tr := &recordingTransport{}
client := &Client{Transport: tr}
url := "http://dummy.faketld/"
client.Get(url) // Note: doesn't hit network
```

那么golang对Io做了哪些优化呢？能实现如此简单的切换呢？

## groutinue 针对io事件的调度

我们这里假设你对groutinue调度已经有一定的了解了。

我们知道,在go中，每个process绑定一个虚拟的machine,而在machine中，是具有一个g0的，g0在本地遍历自己的队列获取g或者从全局队列获取g。
![](https://oscimg.oschina.net/oscnet/up-994aabd6b411dd202955a15a66e4636acff.png)

我们也知道了，在g运行的时候，g会把执行权交给g0进行重新调度，那么在io事件中，g是怎么把事件交还给g0的呢？这时候就牵扯到我们今天的主角----netpoll。


## netpoll

o语言在网络轮询器中使用 I/O 多路复用模型处理 I/O 操作，但是他没有选择最常见的系统调用 `select`。		 `select` 也可以提供 I/O 多路复用的能力，但是使用它有比较多的限制：

- 监听能力有限 — 最多只能监听 1024 个文件描述符，可以通过手动修改limit来改变，但是各方面成本比较大；
- 内存拷贝开销大 — 需要维护一个较大的数据结构存储文件描述符，该结构需要拷贝到内核中；
- 时间复杂度 — 返回准备就绪的事件个数后，需要遍历所有的文件描述符；

golang官方统一封装一个网络事件的poll，和平台无关，为epoll/kqueue/port/AIX/Windows 提供了特定的实现。

- `src/runtime/netpoll_epoll.go`
- `src/runtime/netpoll_kqueue.go`
- `src/runtime/netpoll_solaris.go`
- `src/runtime/netpoll_windows.go`
- `src/runtime/netpoll_aix.go`
- `src/runtime/netpoll_fake.go`

这些模块在不同平台上实现了相同的功能，构成了一个常见的树形结构。编译器在编译 Go 语言程序时，会根据目标平台选择树中特定的分支进行编译

必须实现的方法有：  
```
​netpollinit 初始化网络轮询器，通过 `sync.Once` 和 `netpollInited` 变量保证函数只会调用一次
​netpollopen 监听文件描述符上的边缘触发事件，创建事件并加入监听poll_runtime_pollOpen函数，这个函数将用户态协程的pollDesc信息写入到epoll所在的单独线程，从而实现用户态和内核态的关联。
​netpoll  轮询网络并返回一组已经准备就绪的 Goroutine，传入的参数会决定它的行为：
  - 如果参数小于0，阻塞等待文件就绪
  - 如果参数等于0，非阻塞轮询
  - 如果参数大于0，阻塞定期轮询
​netpollBreak 唤醒网络轮询器，例如：计时器向前修改时间时会通过该函数中断网络轮询器
​netpollIsPollDescriptor  判断文件描述符是否被轮询器使用
```
netpoll中有2个重要的结构体：

```go
//pollCache  
//pollDesc

type pollDesc struct {
	link *pollDesc // in pollcache, protected by pollcache.lock

	// The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
	// This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
	// pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO readiness notification)
	// proceed w/o taking the lock. So closing, everr, rg, rd, wg and wd are manipulated
	// in a lock-free way by all operations.
	// NOTE(dvyukov): the following code uses uintptr to store *g (rg/wg),
	// that will blow up when GC starts moving objects.
	lock    mutex // protects the following fields
	fd      uintptr
	closing bool
	everr   bool      // marks event scanning error happened
	user    uint32    // user settable cookie
	rseq    uintptr   // protects from stale read timers
	rg      uintptr   // pdReady, pdWait, G waiting for read or nil
	rt      timer     // read deadline timer (set if rt.f != nil)
	rd      int64     // read deadline
	wseq    uintptr   // protects from stale write timers
	wg      uintptr   // pdReady, pdWait, G waiting for write or nil
	wt      timer     // write deadline timer
	wd      int64     // write deadline
	self    *pollDesc // storage for indirect interface. See (*pollDesc).makeArg.
}

type pollCache struct {
	lock  mutex
	first *pollDesc
	// PollDesc objects must be type-stable,
	// because we can get ready notification from epoll/kqueue
	// after the descriptor is closed/reused.
	// Stale notifications are detected using seq variable,
	// seq is incremented when deadlines are changed or descriptor is reused.
}
```

- `rseq` 和 `wseq` — 表示文件描述符被重用或者计时器被重置；
- `rg` 和 `wg` — 表示二进制的信号量，可能为 `pdReady`、`pdWait`、等待文件描述符可读或者可写的 Goroutine 以及 `nil`；
- `rd` 和 `wd` — 等待文件描述符可读或者可写的截止日期；
- `rt` 和 `wt` — 用于等待文件描述符的计时器；

golang关于io时间做了很多统一的封装在runtime/netpoll之下（其实调用的是internal/poll包下的）,然后通过internal包下对 runtime包进行调用，internal包下也封装了一个同名的pollDesc对象，不过是一个指针(关于internal有个细节就是这个包是不能被外部调用)：
```go
type pollDesc struct {
	runtimeCtx uintptr
}
```
其实最终都是对runtime底下的调用，只不过封装了一些易用的方法，比如read,write,做了一些抽象化的处理。
```go
func runtime_pollServerInit()  //初始化
func runtime_pollOpen(fd uintptr) (uintptr, int)  //打开
func runtime_pollClose(ctx uintptr)   //关闭
func runtime_pollWait(ctx uintptr, mode int) int //等待
func runtime_pollWaitCanceled(ctx uintptr, mode int) int  //等待并（失败时）退出
func runtime_pollReset(ctx uintptr, mode int) int  //重置状态,复用
func runtime_pollSetDeadline(ctx uintptr, d int64, mode int) //设置读/写超时时间
func runtime_pollUnblock(ctx uintptr)  // 解锁 
func runtime_isPollServerDescriptor(fd uintptr) bool  
// 这里的ctx实际上是一个io fd，不是上下文
// mod 是 r 或者 w  ,io事件毕竟只有有这两种
// d 意义和time.d差不多，就是关于时间的
```

这些方法的具体实现都在runtime下，我们挑几个重要的看看：
```go
//将就绪好得io事件，写入就绪的grotion对列
// netpollready is called by the platform-specific netpoll function.
// It declares that the fd associated with pd is ready for I/O.
// The toRun argument is used to build a list of goroutines to return
// from netpoll. The mode argument is 'r', 'w', or 'r'+'w' to indicate
// whether the fd is ready for reading or writing or both.
//
// This may run while the world is stopped, so write barriers are not allowed.
//go:nowritebarrier
func netpollready(toRun *gList, pd *pollDesc, mode int32) {
	var rg, wg *g
	if mode == 'r' || mode == 'r'+'w' {
		rg = netpollunblock(pd, 'r', true)
	}
	if mode == 'w' || mode == 'r'+'w' {
		wg = netpollunblock(pd, 'w', true)
	}
	if rg != nil {
		toRun.push(rg)
	}
	if wg != nil {
		toRun.push(wg)
	}
}
```

```go
//轮询时调用的方法，如果io就绪了返回ok，如果没就绪，返回flase
// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	// set the gpp semaphore to pdWait
	for {
		old := *gpp
		if old == pdReady {
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}

	// need to recheck error states after setting gpp to pdWait
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
	if waitio || netpollcheckerr(pd, mode) == 0 {
	  //gopark是很重要得一个方法，本质上是让出当前协程执行权，一般是返回到g0让g0重新调度
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
	// be careful to not lose concurrent pdReady notification
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}

//获取到当前io所在的协程，如果协程已关闭，直接返回nil
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	for {
		old := *gpp
		if old == pdReady {
			return nil
		}
		if old == 0 && !ioready {
			// Only set pdReady for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil
		}
		var new uintptr
		if ioready {
			new = pdReady
		}
		if atomic.Casuintptr(gpp, old, new) {
			if old == pdWait {
				old = 0
			}
			return (*g)(unsafe.Pointer(old))
		}
	}
}
```

思考：

1. a、b两个协程，b io阻塞，完成后，一直没有获取到调度权，会出现什么后果。
2. a、b两个协程，b io阻塞，2s time out,但是a一直占用执行权，b一直没有获取到调度权，5s后才获得到，b对使用端已经超时，这时候是超时还是不超时

所以设置的timeout，不一定是真实的io waiting，可能是没有获取到执行权。

## 怎么触发读事件的？

因为写io是我们主动操作的，那么读是怎么进行操作的呢？这是一个被动的状态

首先我们了解一个结构体。golang中所有的网络事件和文件读写都用fd进行标识(位于internal包下)。
```go
// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	fdmu fdMutex

	// System file descriptor. Immutable until Close.
	Sysfd int

	// I/O poller.
	pd pollDesc

	// Writev cache.
	iovecs *[]syscall.Iovec

	// Semaphore signaled when file is closed.
	csema uint32

	// Non-zero if this file has been set to blocking mode.
	isBlocking uint32

	// Whether this is a streaming descriptor, as opposed to a
	// packet-based descriptor like a UDP socket. Immutable.
	IsStream bool

	// Whether a zero byte read indicates EOF. This is false for a
	// message based socket connection.
	ZeroReadIsEOF bool

	// Whether this is a file rather than a network socket.
	isFile bool
}
```
我们看到，fd中关联的pollDesc,通过pollDesc调用了runtime包内部的实现的各种平台的io事件。

当我们进行read操作时（下面是代码截取）
```go
for {
	n, err := ignoringEINTRIO(syscall.Read, fd.Sysfd, p)
	if err != nil {
		n = 0
		if err == syscall.EAGAIN && fd.pd.pollable() {
			if err = fd.pd.waitRead(fd.isFile); err == nil {
				continue
			}
		}
	}
	err = fd.eofError(n, err)
	return n, err
	}
```
会阻塞调用waiteRead方法，方法内部主要就是调用的runtime_pollWait。
```go
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
	errcode := netpollcheckerr(pd, int32(mode))
	if errcode != pollNoError {
		return errcode
	}
	// As for now only Solaris, illumos, and AIX use level-triggered IO.
	if GOOS == "solaris" || GOOS == "illumos" || GOOS == "aix" {
		netpollarm(pd, mode)
	}
	for !netpollblock(pd, int32(mode), false) {
		errcode = netpollcheckerr(pd, int32(mode))
		if errcode != pollNoError {
			return errcode
		}
		// Can happen if timeout has fired and unblocked us,
		// but before we had a chance to run, timeout has been reset.
		// Pretend it has not happened and retry.
	}
	return pollNoError
}
```
这里主要是由netpollblock控制,netpollblock方法我们上面就说过，当io还未就绪的时候，直接释放当前的执行权，否则就是已经课读写的io事件，直接进行读取操作即可。



# 总结

整体流程
	listenStream –> bind&listen&init –> pollDesc.Init -> poll_runtime_pollOpen –>   runtime.netpollopen -> epollctl(EPOLL_CTL_ADD)

画个图来更容易理解，当然，我偷懒是找的图
![](https://oscimg.oschina.net/oscnet/up-68eb3898b759ebc6995bfa3821c7bb75eb1.png)

golang中遇到io事件时，统一对其做了封装，首先建立系统事件（本文主要针对epoll），然后让出cpu（gopark），然后进行协程调度执行其他g。当g io事件完成时，会从epoll进行交互看是否就绪（epoll就绪列表），就绪则pop取出一个g往下执行，未就绪则调度其他g。（其实pop取就绪列表也有一定逻辑，时候延时处理之类的）

runtime/proc.go，
```
// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// The conditions here and in handoffp must agree: if
	// findrunnable would return a G to run, handoffp must start
	// an M.

top:
	_p_ := _g_.m.p.ptr()
	//......
	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}
	//......
}
```
另外在sysmon中，也对netpoll进行了调度。
```
// Always runs without a P, so write barriers are not allowed.
//
//go:nowritebarrierrec
func sysmon() {
	lock(&sched.lock)
	sched.nmsys++
	checkdead()
	unlock(&sched.lock)
	//......
	// poll network if not polled for more than 10ms
	lastpoll := int64(atomic.Load64(&sched.lastpoll))
	if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
		atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
		list := netpoll(0) // non-blocking - returns list of goroutines
		if !list.empty() {
			// Need to decrement number of idle locked M's
			// (pretending that one more is running) before injectglist.
			// Otherwise it can lead to the following situation:
			// injectglist grabs all P's but before it starts M's to run the P's,
			// another M returns from syscall, finishes running its G,
			// observes that there is no work to do and no other running M's
			// and reports deadlock.
			incidlelocked(-1)
			injectglist(&list)
			incidlelocked(1)
		}
	}
	//......
}
```

# 备注

## epoll
epoll是由系统内核单独维护的一个线程,不由go本身维护

## 常量
FD_CLOEXEC用来设置文件的close-on-exec状态标准。 这，emm 就挺难理解得。

## gc
pollDesc是由pollCache进行维护的，并且不受GC监控(persistentalloc方法分配)，所以，在正常情况关于io的操作，我们一定要进行手动关闭，对epoll中的引用对象进行清理(具体实现在poll_runtime_Semrelease)。
```
// Must be in non-GC memory because can be referenced
// only from epoll/kqueue internals.
mem := persistentalloc(n*pdSize, 0, &memstats.other_sys)
for i := uintptr(0); i < n; i++ {
	pd := (*pollDesc)(add(mem, i*pdSize))
	pd.link = c.first
	c.first = pd
}
```

## sysmon
Go 的标准库提供了一种监测应用程序的线程,并帮你 (找寻) 程序可能遇到的瓶颈. 该线程称为sysmon，即系统监视器 (system monitor).在GMP 模型中,这个 (特殊) 线程未链接任何的 P, 这意味着调度器 (scheduler) 没有将其考虑在内, 因此始终处于运行状态.

![](https://oscimg.oschina.net/oscnet/up-9d681a58f5b1cd8d9ac766c99250a9322f7.png)

sysmon线程的作用很广, 主要涉及以下方面:

- 由应用程序创建的计时器 (timers). sysmon线程查看应该在运行却仍在等待执行时间的计时器. 在这种情况下, Go 将查看空闲的 M 和 P 列表, 以便尽可能快地运行它们.
- **网络轮询器和系统调用. 它将运行在网络操作中被阻塞的 goroutine.**
- 垃圾回收器（如果已经很长时间没有运行）. 如果垃圾回收器已经两分钟没有运行,则 sysmon 将强制执行一轮垃圾回收 (GC).
- 长时间运行的 goroutine 的抢占. 任何运行时间超过10 毫秒的 goroutine 都会被抢占, 将运行时间 (running time) 留给其他 goroutine.