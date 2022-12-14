CreateTime:2019-08-27 10:06:07.0

## 什么是ringbuffer
![](https://oscimg.oschina.net/oscnet/3e8179631b936c9d4494292c57c8f4ac8ee.jpg)

嗯，正如名字所说的一样，它是一个环（首尾相接的环），你可以把它用做在不同上下文（线程、协程）间传递数据的buffer。

基本来说，ringbuffer拥有一个固定长度，且每个位置有一个序号，并且是连续的。

随着你不停地填充这个buffer（可能也会有相应的读取），这个序号会一直增长，直到绕过这个环。

一般，ringbuffer都是由数组实现，而由于其在内存上的连续性，因此性能得到了极高的提升。

从数组上看，是这样的，不再是环形。

![](https://oscimg.oschina.net/oscnet/e826240e61f6b07c6899a826bef98ede9e7.jpg)

一般定义的数据结构 

```
type RingBuffer struct{
	buffer []interface{}
	read	uint64  //读的位置
	write	uint64  //写的位置
	size	 uint64  //缓冲区大小
}
```

几大概念：

read == write 时，缓冲区为空

(write + 1) % size == read, 缓冲区满了

## 几大困难点

### 绕回

ringbuffer看上去是一个环，但是实际上是一个数组，写入到数组尾部之后需要绕回到数组首部。但是由于数据包的大小是不定大小，所以到了尾部可能会出现数据分割，包一半在尾部一半在开头，对应的读取的时候数据包一半在尾部，一半在开头需要把它们合并起来。

### 写入过快，释放覆盖

当写入速度大于读取速度时，新写入数据会与未读数据发生覆盖，这样就有两种决策：覆盖未读数据和丢弃新写入数据。

### 避免重复读取

由于ringbuffer是通过覆盖写入数据，并不会删除未读数据，所以就通过ringbuffer中的read来判断是否还有未读数据。

### 重读/重写的需求

假设一个消费场景，消费是耗时的，只有当消费者确认了，这个对象已经被消费掉了，才能被释放掉资源。这时候就需要重读/重写

## 优点

我们使用 Ring Buffer 这种数据结构，是因为它给我们提供了可靠的消息传递特性。

这个理由就足够了，不过它还有一些其他的优点。

首先，Ring Buffer 比链表要快，因为它是数组，而且有一个容易预测的访问模式。

CPU 高速缓存友好 （CPU-cache-friendly）－数据可以在硬件层面预加载到高速缓存，因此 CPU 不需要经常回到主内存 RAM 里去寻找 Ring Buffer 的下一条数据。

Ring Buffer 是一个数组，你可以预先分配内存，并保持数组元素永远有效。这意味着内存垃圾收集（GC）在这种情况下几乎什么也不用做。此外，也不像链表那样每增加一条数据都要创建对象－当这些数据从链表里删除时，这些对象都要被清理掉。

