CreateTime:2019-04-15 15:12:35.0

# 回顾

上文我们分析到了，golang是如何产生一个通道的。

其实很简单，就像所有的高级语言一样，我声明并实现一个对象（虽然go里不叫对象），并给他的分配相应的数据格式和内存空间，这时候对象就存在于我们计算机的内存中了（一般都是堆中）。

# 不可避免的问题

在并发网络的世界里，有个不可避免的问题，就是锁。

# 贴代码（写）

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}
	
	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

## 分析（写）

噢，和我们平常写代码差距不大。

大致就是1.排除错误条件 2.开启日志 3.边界 4. 锁 5. 逻辑 6.释放资源

其实我们应该从锁开始观察就好了。

就是  锁->写数据->释放锁

所以，我们就具体看些数据就行。

## 写数据

emmm,实现

1. 假如找到接受者的话，直接写到接受者队列中

2. 通道缓冲区中有可用空间，将元素写入缓冲区

3. 缓冲区没有可用空间，没有可用空间，将当前 goroutine 加入 send 队列并阻塞，即为同步阻塞。

# 贴代码（读）

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

## 分析（读）

前几个边界，错误判断都差不多

然后读的顺序，好像和写的是有区别

## 读数据

1. 当 send 队列不为空，分两种情况：
 	- 缓存队列为空，直接从 send 队列的sender中接收数据 元素；
 	- 缓存队列不为空，此时只有可能是缓存队列已满，从队列头取出元素，并唤醒 sender 将元素写入缓存队列尾部。由于为环形队列，因此，队列满时只需要将队列头复制给 reciever，同时将 sender 元素复制到该位置，并移动队列头尾索引，不需要移动队列元素。【这就是为什么使用环形队列的原因】
2. 缓冲区（队列）不为空，直接从队列取队头元素，移动头索引。
3. 缓冲区（队列）为空，将 goroutine 加入 recv 队列，并阻塞。

嗯？ 好像就是和写对称反过来而已。

# 锁！！！

无论是读还是写，都是用到了锁，那么，同样是上锁，为什么chan的性能就会高呢？

lock(mutex 互斥锁)字段是这么个含义：

> lock protects all fields in hchan, as well as several fields in sudogs blocked on this channel.
> Do not change another G's status while holding this lock (in particular, do not ready a G), as this can deadlock with stack shrinking.

就是一个锁保护hchan的所有字段，当已经被上锁时，不要去改变任何数据，否则会导致死锁。

至于锁，那是个大难题，且待下回分解！

# 系列文章

[go核心chan.go文件分析解读（一）go是如何构建一个通道的](https://my.oschina.net/lwl1989/blog/3035779 "go核心chan.go文件分析解读（一）go是如何构建一个通道的")

[go核心chan.go文件分析解读（二）go是如何调度通道内容（协程）的](https://my.oschina.net/lwl1989/blog/3037043 "go核心chan.go文件分析解读（二）go是如何调度通道内容（协程）的")

go核心锁的实现