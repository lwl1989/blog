CreateTime:2020-09-14 18:32:50.0

# 引子

golang是一门天然高并发的语言，那既然是并发，就会涉及锁，数据共享以及其原子性操作。今天我们就来看看golang是如何进行数据的原子操作的。

# 详解

### 引子

golang的并发机制是通过协程实现的（我们假设你是对协程有一定了解，没有了解的请自行查阅协程实现机制），协程的基础又是线程，那不可避免的，也会遇到线程同样的问题，也就是我们今天的主人公----锁。

### 锁

为了解决锁的问题，golang的原生库里已经自带了sync包。

一个锁的接口。
```
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}
```

我们来看一下golang的锁如何实现的(这里以互斥锁为例)。

常量以及互斥锁的定义
```
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota
）
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}
```

实现
```
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```
注意啦，很关键啦，这里出现了atomic，翻译出来，也就是原子的。

很简单，我们直译这个方法就是比较并且交换int32的值，也就是m.state。逻辑很简单，如果是0，则交换成1，返回true，否则返回false。

那么重中之重是啥呢？

原子性，是如何实现的。

### 原子性

golang关于锁的原子性的实现都在sync/atomic包下。

不过追踪源码，我们无法窥探其全貌。

```
// BUG(rsc): On x86-32, the 64-bit functions use instructions unavailable before the Pentium MMX.
//
// On non-Linux ARM, the 64-bit functions use instructions unavailable before the ARMv6k core.
//
// On ARM, x86-32, and 32-bit MIPS,
// it is the caller's responsibility to arrange for 64-bit
// alignment of 64-bit words accessed atomically. The first word in a
// variable or in an allocated struct, array, or slice can be relied upon to be
// 64-bit aligned.
```

通过包注释，我们可以理解一下。简而言之：

	64位原子操作的调用者必须确保指针的地址是对齐到8字节的边界

深究源码的话，我们可以发现如下汇编代码：

```
TEXT ·SwapInt32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchg(SB)
```

这里是根据不同的架构调用的不同的汇编系统。

先引入几个概念，看代码之前（收集资料）

	1. FP 伪寄存器是一个虚拟帧指针，用于引用函数参数。
	2. MOVD MOVB等命令之前是优化后的。
	3. ARM64架构中共有34个寄存器,包括31个通用寄存器、SP、PC、CPSR。R0-R30通用寄存器，每个寄存器最多可以存取一个64位大小的数。
		以x0-x30形式使用时是64位的，以w0-w30形式使用时是32位的。
		R0-R7作为函数的参数，R0保存函数的返回值。
		R8寄存器用来保存子程序的返回值，其中，R29又称FP寄存器(frame point)，主要用来保存栈帧（栈底）指针；
		R30又称LR寄存器(link register)，主要用来保存函数返回地址；
		SP：栈顶指针；PC：类似于8086汇编中的IP寄存器，用来记录当前执行的指令地址；
		CPSR：状态寄存器，用来标记运算各种标记
	4.  在多核CPU下，对一个地址的访问可能引起冲突，这个指令解决了冲突，保证原子性(所谓原子操作简单理解就是不能被中断的操作)，是解决多个CPU访问同一内存地址导致冲突的一种机制。通常用于锁，比如spinlock（也就是ldaxr/stlxr）。

以arm64架构为例：

```
TEXT runtime∕internal∕atomic·Xchg64(SB), NOSPLIT, $0-24  //NOSPLIT防止栈分裂 区间【0-24】
again:
	MOVD	ptr+0(FP), R0   //R1 &a  R2  0 R3 1  R0 0
	MOVD	new+8(FP), R1
	LDAXR	(R0), R2        //Load-acquire exclusive register加载并独占
	STLXR	R1, (R0), R3  //store-release  存贮并替换
	CBNZ	R3, again  // R3不为0则跳转到again
	MOVD	R2, ret+16(FP)
	RET
```

可以看出ldaxr和stlxr是保证原子性的关键

这两个命令和最后的movd还暂时有歧义，下回理解后补上。


