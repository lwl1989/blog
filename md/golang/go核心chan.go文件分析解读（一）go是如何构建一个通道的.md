CreateTime:2019-04-11 22:56:37.0

# 原因

今天公众号“一起学golang”发了一篇文章，《深入理解channel,设计+源码》，瞬间提起了我学习的兴趣，好像回到长沙之后，我就暂停了我的学习（事情比较复杂，心情也很复杂）。

都知道go高并发能力特别强，并且是利用他的协程去控制（通常的做法都是goroutine+channel），那我今天就想看看，go是如何实现通道的！

# 预备知识

### 并发

>最早的计算机，每次只能执行一个程序，只有当当前执行的程序结束后才能执行其它程序，在此期间，别的程序都得等着。到后来，计算机运行速度提高了，程序员们发现，单任务运行一旦陷入IO阻塞状态，CPU就没事做了，很是浪费资源，于是就想要同一时间执行那么三五个程序，几个程序一块跑，于是就有了并发。原理就是将CPU时间分片，分别用来运行多个程序，可以看成是多个独立的逻辑流，由于能快速切换逻辑流，看起来就像是大家一块跑的。

### 线程 进程

先不看代码，我们想想，为什么go采用协程+通道模式，现有的并发模型有哪几种。

当然，我们这里只讨论多核模式下的情况（不谈论什么epoll、libevent之类的精巧设计）

1. 多线程并发（线程是进程的一个实体,是**CPU调度和分派的**基本单位,它是比进程更小的能独立运行的基本单位.线程自己基本上不拥有系统资源,只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),**但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源**。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据。）

2. 多进程并发 （进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动,进程是**系统进行资源分配和调度的一个独立单位**。每个进程都有自己的独立内存空间，**不同进程通过进程间通信来通信**。由于**进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大**，但相对比较稳定安全。）

可以看到：

在多核时代，为了更好地利用多核(压榨cpu啊，毕竟硬件的摩尔定律快到头了，不能像以前，加机器！哈哈哈)资源，还是有很多招式的！各位大侠，各显神通。当然，我想表达的东西，也用粗体展现出来了。

代表：java(多线程)、php(多进程)

再说协程吧，其实协程的概念早就有了，go并不是它的发明者，而是它的利用者（协程最初在1963年被提出。），总之，就是更轻量级的任务调度，其中我认为最关键的一项就是，协程全部在用户态就解决了调度问题。


# 源码分析

go的一切源码是皆可见的，因此我们很方便就能看到源码，源码文件位于：

    $GOROOT/src/runtime/chan.go
	
### 结构体

go的channel在源码里定义为hchan

```
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

从命名，我们就能看出大多数的东西了。

其中我们主要要关注的应该是 recvq、sendq、lock

waitq结构如下：
```
type waitq struct {
	first *sudog
	last  *sudog
}
```

源码追踪下：

![](https://oscimg.oschina.net/oscnet/4ac5e485d35bfe7fd84b9094eadfdff77d3.jpg)

可以看出，waitq主要做的是一个对象的编解码，这里涉及runtime.go里定义的一个对象，暂时不讲解（*sudog: sudog represents a g in a wait list, such as for sending/receiving on a channel.)可以把它理解成一个等待的缓冲区（队列）

那么waitq的作用就是获取一个队列的的第一个和最后一个数据（至少从命名上看是这样）

lock，顾名思义，那肯定就是锁了

### go是如何创建一个通道呢？

当你调用如下代码，你可曾想过它是如何被实现：

    channel := make(chan interface{})

首先，chan会被映射到之前我们看到的那个结构体hchan,make自然更容易理解，就是创建嘛，和我们用其他常见的高级语言一样，这里就是创建对象，分配内存，设置GC标志等等一系列操作。

所以，我们这么理解，通道，也是一个对象（JAVA的思想，一切皆对象？）

然后，这个对象有什么呢？

嗯，类型，大小，容积等等。

下面是官方对make(chan interface{})的说明
> 	Channel: The channel's buffer is initialized with the specified
>	buffer capacity. If zero, or the size is omitted, the channel is
>	unbuffered.

其实际实现源码，在如下(1.11.5)：

```
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	if size < 0 || uintptr(size) > maxSliceCap(elem.size) || uintptr(size)*elem.size > maxAlloc-hchanSize {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case size == 0 || elem.size == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.kind&kindNoPointers != 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(uintptr(size)*elem.size, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}
```

源码中我们看到switch,我们只需看默认方式就好了，default

new(hchan)

果然，然后分配了一份内存用于存储即将进来或者出去的goroutine。

这里涉及一个很重要的方法，内存分配。

```
// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer
```

从这里我们可以得知，go小的对象是分配在per-P cache(至于per-P cache是啥，暂且不了解，我们可以把他看成一个比堆还快的区域)中，而大的对象是分配在堆中

### channel调度

channel已经建好好，那么他是如何进行调度的呢？

我们继续追踪源码，发现了这两个东西，噢？这么简单，一个发送一个接收？

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool)
```

那不正好对应

```
channel <-
<- channel
```

至于怎么调度，且待下回分解，go是调度通道内容的


# 系列文章

[go核心chan.go文件分析解读（一）go是如何构建一个通道的](https://my.oschina.net/lwl1989/blog/3035779 "go核心chan.go文件分析解读（一）go是如何构建一个通道的")

[go核心chan.go文件分析解读（二）go是如何调度通道内容（协程）的](https://my.oschina.net/lwl1989/blog/3037043 "go核心chan.go文件分析解读（二）go是如何调度通道内容（协程）的")

go核心锁的实现

