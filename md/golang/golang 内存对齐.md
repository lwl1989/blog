CreateTime:2022-01-05 21:44:49.0

# 内存对齐是什么？

> 内存对齐”应该是编译器的“管辖范围”。编译器为程序中的每个“数据单元”安排在适当的位置上。但是C语言的一个特点就是太灵活，太强大，它允许你干预“内存对齐”。如果你想了解更加底层的秘密，“内存对齐”对你就不应该再模糊了。

百度百科其实讲的太晦涩

[张彦飞的内存对齐的理解](https://zhuanlan.zhihu.com/p/83449008 "张彦飞的内存对齐的理解") 看这个可以更详细的理解

## 为什么要内存对齐

- 平台原因(移植原因)：不是所有的硬件平台都能访问任意地址上的任意数据的；某些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。
- 性能原因：数据结构(尤其是栈)应该尽可能地在自然边界上对齐。原因在于，为了访问未对齐的内存，处理器需要作两次内存访问；而对齐的内存访问仅需要一次访问。


# golang内存对齐

```
type A struct {
	a uint8
	b uint64
	c uint32
	d uint8
}

type B struct {
	a uint8
	d uint8
	c uint32
	b uint64
}

func TestMemory(t *testing.T)  {
	a := A{}
	b := B{}

	fmt.Println(fmt.Sprintf("a size: %d", unsafe.Sizeof(a)))
	fmt.Println(fmt.Sprintf("b size: %d", unsafe.Sizeof(b)))
}
```
同样一个结构体，会产生什么样的情况呢？

![](https://oscimg.oschina.net/oscnet/up-c14ba40fdbb589d3f0df456042ea1008833.png)

这是见证奇迹的时刻吗？

```
type C struct {
	a uint8
	d uint8
	c uint32
	b uint64
	e struct{}
}

func TestMemory1(t *testing.T)  {
	c := C{}
	fmt.Println(unsafe.Offsetof(c.e))
	fmt.Println(unsafe.Sizeof(c.e))
	fmt.Println(unsafe.Sizeof(c))
}
```

![](https://oscimg.oschina.net/oscnet/up-e002619faad36e2b88ab6e146da13eea28c.png)

是不是好奇了？

明明我们的内存是对齐的，而按照go对空结构体的大小定义是0，应该C结构体的大小是16才对，为什么是24呢？offset 不是16吗？ 16+0 = 24？

这里要引入一个新的名词，padding

## padding

每个数据类型在数组里占多少位我们都清楚，那为什么结果和预期不一致呢？

上面2个例子中，我们可以看到，同一个结构体（内容相同的）会产生不同大小的结果，一个空结构体，会额外占用8个字节的位置。这个位置就是由padding填充的。

[issue](https://github.com/golang/go/issues/9401 "issue"),另外提及一个关于空结构体的issue。这块就不详细赘述了，空结构体为什么要分配内存的原因。

以整型来举例

Go的整数类型一共有10个，其中计算架构相关的整数类型有两个: 有符号的整数类型 int, 无符号的整数类型 uint。在不同计算架构的计算机上，它们体现的宽度(存储某个类型的值所需要的空间)是不一样的。空间的单位可以是bit也可以是字节byte。

![](https://oscimg.oschina.net/oscnet/up-eb9f00c63a23917130d0d2604053cbc9c7f.png)
![](https://oscimg.oschina.net/oscnet/up-ffec974e7244a76db1f550d8d8317396aca.png)

总结出来就是这样：
![](https://oscimg.oschina.net/oscnet/up-fb2873780af29982b1af0857dcbecbdaa35.png)
struct 的对齐是：如果类型 t 的对齐保证是 n，那么类型 t 的每个值的地址在运行时必须是 n 的倍数。即 uintptr(unsafe.Pointer(&x)) % unsafe.Alignof(x) == 0
# span

go的内存分配，首先是按照sizeclass划分span，然后每个span中的page又分成一个个小格子(大小相同的对象object)：

span是golang内存管理的基本单位，是由一片连续的8KB(golang page的大小)的页组成的大块内存。

每个span管理指定规格（以golang 中的 page为单位）的内存块，内存池分配出不同规格的内存块就是通过span体现出来的，应用程序创建对象就是通过找到对应规格的span来存储的，下面是 mspan结构中的主要部分。

```
//go:notinheap
type mspan struct {
   next *mspan     //链表下一个span地址
   prev *mspan     // 链表前一个span地址
   list *mSpanList // 链表地址

   startAddr uintptr // 该span在arena区域的起始地址
   npages    uintptr // 该span占用arena区域page的数量

   manualFreeList gclinkptr // 空闲对象列表

   freeindex uintptr//freeindex是0到nelems之间的位置索引,标记下一个空对象索引

   nelems uintptr // 管理的对象数

   allocCache uint64   //从freeindex开始的位标记
   allocBits  *gcBits //该mspan中对象的位图
   gcmarkBits *gcBits //该mspan中标记的位图,用于垃圾回收
   sweepgen    uint32 //扫描计数值，用户与mheap的sweepgen比较，根据差值确定该span的扫描状态

   allocCount  uint16     // 已分配的对象的个数
   spanclass   spanClass  // span分类
   state       mSpanState // mspaninuse etc
   needzero    uint8      // 分配之前需要置零

   scavenged   bool       // 标记是否内存已经被系统回收，大对象会用到
   elemsize    uintptr    // 对象的大小
   unusedsince int64      // 空闲状态开始的纳秒值时间戳，用于系统内存释放
   limit       uintptr    // 申请大对象内存块会用到，mspan的数据截止位置
   speciallock mutex         // guards 专用锁
   specials    *special      //按偏移量排序的特殊记录的链接列表。
}
```

每个mspan按照它自身的属性Size Class的大小分割成若干个object，每个object可存储一个对象。

![](https://oscimg.oschina.net/oscnet/up-34cb923527ee6bf71642c5e5cd802b139c3.png)

并且会使用一个位图来标记其尚未使用的object。属性Size Class决定object大小，而mspan只会分配给和object尺寸大小接近的对象，当然，对象的大小要小于object大小。

# 总结

1. 内存对齐是为了让 cpu 更高效访问内存中数据。

2. struct 内字段如果填充过多，可以尝试重排，使字段排列更紧密，减少内存浪费

3. 零大小字段要避免作为 struct 最后一个字段，会有内存浪费，32 位系统上对 64 位字的原子访问要保证其是 8bytes 对齐的。

任何复杂的系统都是由很多细节优化去保证的性能，这个padding其实是在编译阶段是优化的，在runtime中没有处理过。我们在编写的过程中稍微注意一些细节，就能给我们的系统节约大量的内存（其实也节约了时间，减少了malloc 和 unmalloc）。


