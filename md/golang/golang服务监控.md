CreateTime:2021-12-21 17:08:53.0

> 以linux服务器为例子，正常情况下，我们要获取服务的内存信息一般都是通过相关的命令，例如top、ps等命令。然而这些监控内存使用情况的方法，一般需要编写脚本，执行脚本后将执行结果发送给对应的监控服务，从而达到监控的效果。

> 本文代码基于1.14阅读。至于原因，肯定是因为某种需求。

# 问题

常规的监控方式会有什么问题呢？

内存飙升，但是我们根本不知道为什么，只能通过经验去分析我们的代码，瓶颈在哪，哪里有溢出，怎么优化之类的。

# pprof

golang在runtime包底下提供pprof的工具以方便我们的使用。


## 安装

	go get -u github.com/google/pprof

## 分析工具(命令行)

go tool pprof 是命令行指令，用于分析 Profiling 数据，源数据可以是 http 地址，也可以是已经 dump 下当 profile 文件；查看模式可以命令行交互模式，也可以是浏览器模式（-http 参数）。

两种应用:

- web服务型应用 _ "net/http/pprof" 包，专用于采集 web 服务 运行数据的分析。即在运行的服务中通过 API 调用取数据。

	服务型应用场景中因为应用要一直提供服务。所以 pprof 是通过 API 访问来获取，pprof 使用了默认的 http.DefaultServeMux 挂在这些 API 接口。开发者也可以手动注册路由规则挂载到指定的路由API。

- 工具包型应用 "runtime/pprof" 包，专用于采集应用程序运行数据的分析。通过代码手动添加收集命令。

	工具型应用是一个提供特定api go package，需要开发者进行编码写入文件。也可以将信息封装到接口方便调用，比如：

	要进行 CPU Profiling，则调用 pprof.StartCPUProfile(w io.Writer) 写入到 w 中，停止时调用 StopCPUProfile()；
	要获取内存数据，直接使用 pprof.WriteHeapProfile(w io.Writer) 函数则可。

## 支持的分析内容

```
CPU 分析（profile）: 你可以在 url 上用 seconds 参数指定抽样持续时间（默认 30s），你获取到概览文件后可以用 go tool pprof 命令调查这个概览
内存分配（allocs）: 所有内存分配的抽样
阻塞（block）: 堆栈跟踪导致阻塞的同步原语
命令行调用（cmdline）: 命令行调用的程序
goroutine: 当前 goroutine 的堆栈信息
堆（heap）: 当前活动对象内存分配的抽样，完全也可以指定 gc 参数在对堆取样前执行 GC
互斥锁（mutex）: 堆栈跟踪竞争状态互斥锁的持有者
系统线程的创建（threadcreate）: 堆栈跟踪系统新线程的创建
trace: 追踪当前程序的执行状况. 可以用 seconds 参数指定抽样持续时间,获取到 trace 概览后可以用 go tool pprof 命令调查这个 trace
```
## 使用

### http

要使用http服务,直接使用官方示例即可

```
// To use pprof, link this package into your program:
//	import _ "net/http/pprof"
//
// If your application is not already running an http server, you
// need to start one. Add "net/http" and "log" to your imports and
// the following code to your main function:
//
// 	go func() {
// 		log.Println(http.ListenAndServe("localhost:6060", nil))
// 	}()
```

pprof默认注册了如下路由，其实我们也可以修改path
```
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
```

直接访问相关path就能得到结果

![image.png](http://ttc-tal.oss-cn-beijing.aliyuncs.com/1640250348/image.png)

通常我们会配合Go tool pprof进行数据采集再去进行分析。

### 手动编码

举个列子，获取goroutine数量。这是官方的示例。
```
func TestGoroutineCounts(t *testing.T) {
	// Setting GOMAXPROCS to 1 ensures we can force all goroutines to the
	// desired blocking point.
	defer runtime.GOMAXPROCS(runtime.GOMAXPROCS(1))

	c := make(chan int)
	for i := 0; i < 100; i++ {
		switch {
		case i%10 == 0:
			go func1(c)
		case i%2 == 0:
			go func2(c)
		default:
			go func3(c)
		}
		// Let goroutines block on channel
		for j := 0; j < 5; j++ {
			runtime.Gosched()
		}
	}

	var w bytes.Buffer
	goroutineProf := Lookup("goroutine")

	// Check debug profile
	goroutineProf.WriteTo(&w, 1)
	prof := w.String()

	if !containsInOrder(prof, "\n50 @ ", "\n40 @", "\n10 @", "\n1 @") {
		t.Errorf("expected sorted goroutine counts:\n%s", prof)
	}

	// Check proto profile
	w.Reset()
	goroutineProf.WriteTo(&w, 0)
	p, err := profile.Parse(&w)
	if err != nil {
		t.Errorf("error parsing protobuf profile: %v", err)
	}
	if err := p.CheckValid(); err != nil {
		t.Errorf("protobuf profile is invalid: %v", err)
	}
	if !containsCounts(p, []int64{50, 40, 10, 1}) {
		t.Errorf("expected count profile to contain goroutines with counts %v, got %v",
			[]int64{50, 40, 10, 1}, p)
	}

	close(c)

	time.Sleep(10 * time.Millisecond) // let goroutines exit
}
```
可以看出，作为开发者的我们，只需要简单的引入一个接口类型io.Writer的实现对象，将其写入部门修改即可（当然我们可以做很多扩展，比如落地、报警，结合数据限流、熔断之类的）。


## 支持的分析内容

```
CPU 分析（profile）: 你可以在 url 上用 seconds 参数指定抽样持续时间（默认 30s），你获取到概览文件后可以用 go tool pprof 命令调查这个概览
内存分配（allocs）: 所有内存分配的抽样
阻塞（block）: 堆栈跟踪导致阻塞的同步原语
命令行调用（cmdline）: 命令行调用的程序
goroutine: 当前 goroutine 的堆栈信息
堆（heap）: 当前活动对象内存分配的抽样，完全也可以指定 gc 参数在对堆取样前执行 GC
互斥锁（mutex）: 堆栈跟踪竞争状态互斥锁的持有者
系统线程的创建（threadcreate）: 堆栈跟踪系统新线程的创建
trace: 追踪当前程序的执行状况. 可以用 seconds 参数指定抽样持续时间,获取到 trace 概览后可以用 go tool pprof 命令调查这个 trace
```

## 实现原理

pprof这么强大，那么pprof是怎么实现的呢？带着这个问题去阅读pprof源码

```go
// profiles records all registered profiles.
var profiles struct {
	mu sync.Mutex
	m  map[string]*Profile
}

var goroutineProfile = &Profile{
	name:  "goroutine",
	count: countGoroutine,
	write: writeGoroutine,
}

var threadcreateProfile = &Profile{
	name:  "threadcreate",
	count: countThreadCreate,
	write: writeThreadCreate,
}

var heapProfile = &Profile{
	name:  "heap",
	count: countHeap,
	write: writeHeap,
}

var allocsProfile = &Profile{
	name:  "allocs",
	count: countHeap, // identical to heap profile
	write: writeAlloc,
}

var blockProfile = &Profile{
	name:  "block",
	count: countBlock,
	write: writeBlock,
}

var mutexProfile = &Profile{
	name:  "mutex",
	count: countMutex,
	write: writeMutex,
}

func lockProfiles() {
	profiles.mu.Lock()
	if profiles.m == nil {
		// Initial built-in profiles.
		//所以，这些定义不是和上面的支持的命令一样吗？
		profiles.m = map[string]*Profile{
			"goroutine":    goroutineProfile,
			"threadcreate": threadcreateProfile,
			"heap":         heapProfile,
			"allocs":       allocsProfile,
			"block":        blockProfile,
			"mutex":        mutexProfile,
		}
	}
}
```

### goroutine

协程的状态是遍历整个Stack获取的，至于协程数量就比较好获取，gcount()直接可以得到
```go
// 如果不是取全部，则只取当前g的栈信息
func Stack(buf []byte, all bool) int {
	if all {
		stopTheWorld("stack trace") //如果遍历所有，是需要stw的，慎用
	}

	n := 0
	if len(buf) > 0 {
		gp := getg()
		sp := getcallersp()
		pc := getcallerpc()
		systemstack(func() { //systemstack是go内部一个及其重要的方法，可以理解成你当前切换到了GMP模型的M上，我们的业务逻辑通常只会运行在G上
			g0 := getg()
			g0.m.traceback = 1
			g0.writebuf = buf[0:0:len(buf)]
			goroutineheader(gp)
			traceback(pc, sp, 0, gp)
			if all {
				tracebackothers(gp)
			}
			g0.m.traceback = 0
			n = len(g0.writebuf)
			g0.writebuf = nil
		})
	}

	if all {
		startTheWorld()
	}
	return n
}
```

### threadcreate

线程这个比较干脆，直接遍历链表获取，数量也就是链表长度
```go
func ThreadCreateProfile(p []StackRecord) (n int, ok bool) {
	first := (*m)(atomic.Loadp(unsafe.Pointer(&allm))) //allm存放一个全局单向链表指针
	for mp := first; mp != nil; mp = mp.alllink {
		n++
	}
	if n <= len(p) {
		ok = true
		i := 0
		for mp := first; mp != nil; mp = mp.alllink {
			p[i].Stack0 = mp.createstack
			i++
		}
	}
	return
}
```

### 


### 堆heap，以及内存使用情况allocs

在内存方向，golang都使用了同一个结构体进行存储(runtime.MemStats)：

```
runtime.MemStats这个结构体包含的字段比较多，但是大多都很有用：
Alloc uint64 //golang语言框架堆空间分配的字节数
TotalAlloc uint64 //从服务开始运行至今分配器为分配的堆空间总 和，只有增加，释放的时候不减少
Sys uint64 //服务现在系统使用的内存
Lookups uint64 //被runtime监视的指针数
Mallocs uint64 //服务malloc heap objects的次数
Frees uint64 //服务回收的heap objects的次数
HeapAlloc uint64 //服务分配的堆内存字节数
HeapSys uint64 //系统分配的作为运行栈的内存
HeapIdle uint64 //申请但是未分配的堆内存或者回收了的堆内存（空闲）字节数
HeapInuse uint64 //正在使用的堆内存字节数
HeapReleased uint64 //返回给OS的堆内存，类似C/C++中的free。
HeapObjects uint64 //堆内存块申请的量
StackInuse uint64 //正在使用的栈字节数
StackSys uint64 //系统分配的作为运行栈的内存
MSpanInuse uint64 //用于测试用的结构体使用的字节数
MSpanSys uint64 //系统为测试用的结构体分配的字节数
MCacheInuse uint64 //mcache结构体申请的字节数(不会被视为垃圾回收)
MCacheSys uint64 //操作系统申请的堆空间用于mcache的字节数
BuckHashSys uint64 //用于剖析桶散列表的堆空间
GCSys uint64 //垃圾回收标记元信息使用的内存
OtherSys uint64 //golang系统架构占用的额外空间
NextGC uint64 //垃圾回收器检视的内存大小
LastGC uint64 // 垃圾回收器最后一次执行时间。
PauseTotalNs uint64 // 垃圾回收或者其他信息收集导致服务暂停的次数。
PauseNs [256]uint64 //一个循环队列，记录最近垃圾回收系统中断的时间
PauseEnd [256]uint64 //一个循环队列，记录最近垃圾回收系统中断的时间开始点。
NumForcedGC uint32 //服务调用runtime.GC()强制使用垃圾回收的次数。
GCCPUFraction float64 //垃圾回收占用服务CPU工作的时间总和。如果有100个goroutine，垃圾回收的时间为1S,那么就占用了100S。
BySize //内存分配器使用情况
```

在pprof中是调用的writeHeapInternal,进行读写渲染
```go
func writeHeapInternal(w io.Writer, debug int, defaultSampleType string) error {
	var s *runtime.MemStats
	// do any more
	fmt.Fprintf(w, "\n# runtime.MemStats\n")
	// print any more
	fmt.Fprintf(w, "# DebugGC = %v\n", s.DebugGC)
}
```

runtime.ReadMemStats方法是需要stw的，所以尽量不要在线上调用
```go
func ReadMemStats(m *MemStats) {
	stopTheWorld("read mem stats")

	systemstack(func() {
		readmemstats_m(m)
	})

	startTheWorld()
}
```

### 阻塞代码块(block)

block代码块信息一样，也是通过了一个全局的bbuckets指针存放信息。
```go
func BlockProfile(p []BlockProfileRecord) (n int, ok bool) {
	lock(&proflock)
	for b := bbuckets; b != nil; b = b.allnext {
		n++
	}
	if n <= len(p) {
		ok = true
		for b := bbuckets; b != nil; b = b.allnext {
			bp := b.bp()
			r := &p[0]
			r.Count = bp.count
			r.Cycles = bp.cycles
			if raceenabled {
				racewriterangepc(unsafe.Pointer(&r.Stack0[0]), unsafe.Sizeof(r.Stack0), getcallerpc(), funcPC(BlockProfile))
			}
			if msanenabled {
				msanwrite(unsafe.Pointer(&r.Stack0[0]), unsafe.Sizeof(r.Stack0))
			}
			i := copy(r.Stack0[:], b.stk())
			for ; i < len(r.Stack0); i++ {
				r.Stack0[i] = 0
			}
			p = p[1:]
		}
	}
	unlock(&proflock)
	return
}
```

肯定很好奇block代码块是怎么产生的。其实和我们的CURD一样，就是CREATE和UPDATE阶段生成的(stkbucket方法内追加到链表尾部)。

```go
//保存上下文  update
func saveblockevent(cycles int64, skip int, which bucketType) {
	gp := getg()
	//...
	b := stkbucket(which, 0, stk[:nstk], true)
    //...
}

//创建新的 create
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	//...
	
	mp := acquirem()
	profilealloc(mp, x, size) // => mProf_Malloc => stkbucket
	releasem(mp)
	//
}
```

### 同步代码块（mutex）

同步代码块的记录方式和block类似，都是通过一个链表进行存储，写入方法也一样（stkbucket方法）。
```go
func MutexProfile(p []BlockProfileRecord) (n int, ok bool) {
	lock(&proflock)
	for b := xbuckets; b != nil; b = b.allnext {
		n++
	}
	if n <= len(p) {
		ok = true
		for b := xbuckets; b != nil; b = b.allnext {
			bp := b.bp()
			r := &p[0]
			r.Count = int64(bp.count)
			r.Cycles = bp.cycles
			i := copy(r.Stack0[:], b.stk())
			for ; i < len(r.Stack0); i++ {
				r.Stack0[i] = 0
			}
			p = p[1:]
		}
	}
	unlock(&proflock)
	return
}
```

stkbucket看来是一个比较重要的方法
```
type bucket struct {
	next    *bucket
	allnext *bucket
	typ     bucketType // memBucket or blockBucket (includes mutexProfile)
	hash    uintptr
	size    uintptr
	nstk    uintptr
}
func stkbucket(typ bucketType, size uintptr, stk []uintptr, alloc bool) *bucket {
	if buckhash == nil {
		buckhash = (*[buckHashSize]*bucket)(sysAlloc(unsafe.Sizeof(*buckhash), &memstats.buckhash_sys))
		if buckhash == nil {
			throw("runtime: cannot allocate memory")
		}
	}

	// Hash stack.
	var h uintptr
	// hash 计算...

	i := int(h % buckHashSize)
	for b := buckhash[i]; b != nil; b = b.next {
	//如果已经存在最直接返回
		if b.typ == typ && b.hash == h && b.size == size && eqslice(b.stk(), stk) {
			return b
		}
	}

	if !alloc {
		return nil
	}

	// Create new bucket.
	b := newBucket(typ, len(stk))
	copy(b.stk(), stk)  //这里其实我略带疑惑，全局copy一个，会不会造成内存过多？
	b.hash = h
	b.size = size
	b.next = buckhash[i]
	buckhash[i] = b
	//重新赋值相应的头指针
	if typ == memProfile {
		b.allnext = mbuckets 
		mbuckets = b
	} else if typ == mutexProfile {
		b.allnext = xbuckets
		xbuckets = b
	} else {
		b.allnext = bbuckets
		bbuckets = b
	}
	return b
}
```
![](https://oscimg.oschina.net/oscnet/up-b5fbccd38f47be2f83ddf60e591893a56c7.png)

## 汇总

通过阅读源码，我们可以发现，pprof包的具体实现都在runtime包之上做了封装。尽管 Go 编译器产生的是本地可执行代码，但是大部分信息仍旧运行在 Go 的 runtime，并且可以寻踪匿迹，有完整的trace。

要注意的是：

	很多地方都涉及到stw，线上需要慎重调用。

对于现在GO最主流的监控开源组件prometheus，其内部也是大量的使用runtime包底下的方法进行监控，还有一部分是调用的syscall调用的系统内部方法(这个不在本文讨论中)。



