CreateTime:2021-04-20 19:08:14.0

# 现在面试都这么直接的嘛？

面试难如狗，肝不过年轻人怎么办，只能多总结。

# 闲聊

### MAP结构
Map的实现主要有两种方式：哈希表（hash table）和搜索树（search tree）。例如Java中的hashMap是基于哈希表实现，而C++中的Map是基于一种平衡搜索二叉树——红黑树而实现的。以下是不同实现方式的时间复杂度对比。

![](https://oscimg.oschina.net/oscnet/up-11b6c8fc6af8365e7bb8e4a03aaa56165d2.png)

可以看到，对于元素查找而言，二叉搜索树的平均和最坏效率都是O(log n)，哈希表实现的平均效率是O(1)，但最坏情况下能达到O(n)，不过如果哈希表设计优秀，最坏情况基本不会出现（所以，读者不想知道Go是如何设计的Map吗）。另外二叉搜索树返回的key是有序的，而哈希表则是乱序。

### 哈希表
由于Go中map的基于哈希表（也被称为散列表）实现，本文不探讨搜索树的map实现。以下是Go官方博客对map的说明。

One of the most useful data structures in computer science is the hash table. Many hash table implementations exist with varying properties, but in general they offer fast lookups, adds, and deletes. Go provides a built-in map type that implements a hash table.

学习哈希表首先要理解两个概念：哈希函数和哈希冲突。

#### 哈希函数
哈希函数（常被称为散列函数）是可以用于将任意大小的数据映射到固定大小值的函数，常见的包括MD5、SHA系列等。


一个设计优秀的哈希函数应该包含以下特性：

- 均匀性：一个好的哈希函数应该在其输出范围内尽可能均匀地映射，也就是说，应以大致相同的概率生成输出范围内的每个哈希值。
- 效率高：哈希效率要高，即使很长的输入参数也能快速计算出哈希值。
- 可确定性：哈希过程必须是确定性的，这意味着对于给定的输入值，它必须始终生成相同的哈希值。
- 雪崩效应：微小的输入值变化也会让输出值发生巨大的变化。
- 不可逆：从哈希函数的输出值不可反向推导出原始的数据。

#### 哈希冲突
重复一遍，哈希函数是将任意大小的数据映射到固定大小值的函数。那么，我们可以预见到，即使哈希函数设计得足够优秀，几乎每个输入值都能映射为不同的哈希值。但是，当输入数据足够大，大到能超过固定大小值的组合能表达的最大数量数，冲突将不可避免！（参见抽屉原理）
![](https://oscimg.oschina.net/oscnet/up-fca23fff71d0fb9cc51b6a1675031a67093.png)

抽屉原理：桌上有十个苹果，要把这十个苹果放到九个抽屉里，无论怎样放，我们会发现至少会有一个抽屉里面放不少于两个苹果。抽屉原理有时也被称为鸽巢原理。

# map


### 源码解析

嚄  你也是今天的主角了

数据结构？

好像和java很像，数组链表。多说无益，没什么比源码更清楚。

```
// A header for a Go map.
type hmap struct {
    count     int // 代表哈希表中的元素个数，调用len(map)时，返回的就是该字段值。
    flags     uint8 // 状态标志，下文常量中会解释四种状态位含义。
    B         uint8  // buckets（桶）的对数log_2（哈希表元素数量最大可达到装载因子*2^B）
    noverflow uint16 // 溢出桶的大概数量。
    hash0     uint32 // 哈希种子。

    buckets    unsafe.Pointer // 指向buckets数组的指针，数组大小为2^B，如果元素个数为0，它为nil。
    oldbuckets unsafe.Pointer // 如果发生扩容，oldbuckets是指向老的buckets数组的指针，老的buckets数组大小是新的buckets的1/2。非扩容状态下，它为nil。
    nevacuate  uintptr        // 表示扩容进度，小于此地址的buckets代表已搬迁完成。

    extra *mapextra // 这个字段是为了优化GC扫描而设计的。当key和value均不包含指针，并且都可以inline时使用。extra是指向mapextra类型的指针。
}
```
然后还有个bmap结构这，每个bucket里面存储了kv对。buckets是一个指针，指向实际存储的bucket数组的首地址。 bucket的结构体如下：
```
type bmap struct {
	tophash [bucketCnt]uint8  //一个8个长度的uint8组成 
}
```
其实上面这个数据结构并不是 golang runtime 时的结构，在编译时候编译器会给它动态创建一个新的结构，如下：
```
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

bmap 就是我们常说的“bucket”结构，每个 bucket 里面最多存储 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置,ps:一类的计算）。

对应一个图的话

![](https://oscimg.oschina.net/oscnet/up-24881e71cf5f83352a531a849e0ec1693db.png)


### 使用

golang使用，直接用make的。


```
make(map[K]V)
make(map[K]V, len) //我靠，今天踩知道map可以分配初始大小，是我失策了
```

make函数实际上会被编译器定位到调用 runtime.makemap()，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等等。
```
// 如果编译器认为map和第一个bucket可以直接创建在栈上，h和bucket可能都是非空
// 如果h != nil，那么map可以直接在h中创建
// 如果h.buckets != nil，那么h指向的bucket可以作为map的第一个bucket使用
func makemap(t *maptype, hint int, h *hmap) *hmap {
  // math.MulUintptr返回hint与t.bucket.size的乘积，并判断该乘积是否溢出。
    mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
// maxAlloc的值，根据平台系统的差异而不同，具体计算方式参照src/runtime/malloc.go
    if overflow || mem > maxAlloc {
        hint = 0
    }

// initialize Hmap
    if h == nil {
        h = new(hmap)
    }
  // 通过fastrand得到哈希种子
    h.hash0 = fastrand()

    // 根据输入的元素个数hint，找到能装下这些元素的B值
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 分配初始哈希表
  // 如果B为0，那么buckets字段后续会在mapassign方法中lazily分配
    if h.B != 0 {
        var nextOverflow *bmap
    // makeBucketArray创建一个map的底层保存buckets的数组，它最少会分配h.B^2的大小。
        h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
        if nextOverflow != nil {
    h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }

    return h
}
```
根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 hmap 中的不同字段引用而已。

注意，这个函数返回的结果：*hmap 是一个指针，而我们之前讲过的 makeslice 函数返回的是 Slice 结构体对象。这也是 makemap 和 makeslice 返回值的区别所带来一个不同点：当 map 和 slice 作为函数参数时，在函数参数内部对 map 的操作会影响 map 自身；而对 slice 却不会（之前讲 slice 的文章里有讲过）。

主要原因：一个是指针（*hmap），一个是结构体（slice）。Go 语言中的函数传参都是值传递，在函数内部，参数会被 copy 到本地。*hmap指针 copy 完之后，仍然指向同一个 map，因此函数内部对 map 的操作会影响实参。而 slice 被 copy 后，会成为一个新的 slice，对它进行的操作不会影响到实参。

### hash函数

哈希函数的算法与key的类型一一对应的。根据 key 的类型， maptype结构体的 key字段的alg 字段会被设置对应类型的 hash 和 equal 函数。

在初始化go程序运行环境时（src/runtime/proc.go中的schedinit），就需要通过alginit方法完成对哈希的初始化。


### hash key桶的分配

对于 hashmap 来说，最重要的就是根据key定位实际存储位置。key 经过哈希计算后得到哈希值，哈希值是 64 个 bit 位（针对64位机）。根据hash值的最后B个bit位来确定这个key落在哪个桶。如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。

suppose，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

	10010111 | 000011110110110010001111001010100010010110010101010 │ 00110

![](https://oscimg.oschina.net/oscnet/up-cbaec1e8bb3d3e84660578710d6abd43974.png)

定位key，如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。(这里需要遍历bucket数组中某个槽位的bucket链表的所有bucket)

![](https://oscimg.oschina.net/oscnet/up-958f3953b9a51539ec512412a9efe181d1a.png)


# 扩容

使用 key 的 hash 值可以快速定位到目标 key，然而，随着向 map 中添加的 key 越来越多，key 发生碰撞的概率也越来越大。bucket 中的 8 个 cell 会被逐渐塞满，查找、插入、删除 key 的效率也会越来越低。最理想的情况是一个 bucket 只装一个 key，这样，就能达到 O(1) 的效率，但这样空间消耗太大，用空间换时间的代价太高。

Go 语言采用一个 bucket 里装载 8 个 key，定位到某个 bucket 后，还需要再定位到具体的 key，这实际上又用了时间换空间。

当然，这样做，要有一个度，不然所有的 key 都落在了同一个 bucket 里，直接退化成了链表，各种操作的效率直接降为 O(n)，是不行的。

因此，需要有一个指标来衡量前面描述的情况，这就是装载因子。Go 源码里这样定义 装载因子：

	loadFactor := count / (2^B)

count 就是 map 的元素个数，2^B 表示 bucket 数量。

再来说触发 map 扩容的时机：在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：

1. 加载因子超过阈值，源码里定义的阈值是 6.5。

2. overflow 的 bucket 数量过多，这有两种情况：
	- 当 B 大于15时，也就是 bucket 总数大于 2^15 时，如果overflow的bucket数量大于2^15，就触发扩容。
	- 当B小于15时，如果overflow的bucket数量大于2^B 也会触发扩容。

### 如何降低map扩容的影响

因为map扩容是很消耗性能的，（桶的新建、复制），因此我们要尽量减少扩容，初始化的时候对数据进行预期分配。

# sync.Map如何降低锁的粒度

从map设计可以知道，它并不是一个并发安全的数据结构。同时对map进行读写时，程序很容易出错。因此，要想在并发情况下使用map，请加上锁（sync.Mutex或者sync.RwMutex）。其实，Go标准库中已经为我们实现了并发安全的map——sync.Map。

### sync.Map介绍
go 1.9 官方提供了sync.Map 来优化线程安全的并发读写的map。该实现也是基于内置map关键字来实现的。

这个实现类似于一个线程安全的 map[interface{}]interface{} . 这个map的优化主要适用了以下场景：

（1）给定key的键值对只写了一次，但是读了很多次，比如在只增长的缓存中；
（2）当多个goroutine读取、写入和覆盖的key值不相交时。

在这两种情况下，使用Map可能比使用单独互斥锁或RWMutex的Go Map大大减少锁争用。

对于其余情况最好还是使用RWMutex保证线程安全。
```
// 封装的线程安全的map
type Map struct {
	// lock
	mu Mutex

	// 实际是readOnly这个结构
	// 一个只读的数据结构，因为只读，所以不会有读写冲突。
	// readOnly包含了map的一部分数据，用于并发安全的访问。(冗余，内存换性能)
	// 访问这一部分不需要锁。
	read atomic.Value // readOnly

	// dirty数据包含当前的map包含的entries,它包含最新的entries(包括read中未删除的数据,虽有冗余，但是提升dirty字段为read的时候非常快，不用一个一个的复制，而是直接将这个数据结构作为read字段的一部分),有些数据还可能没有移动到read字段中。
	// 对于dirty的操作需要加锁，因为对它的操作可能会有读写竞争。
	// 当dirty为空的时候， 比如初始化或者刚提升完，下一次的写操作会复制read字段中未删除的数据到这个数据中。
	dirty map[interface{}]*entry

	// 当从Map中读取entry的时候，如果read中不包含这个entry,会尝试从dirty中读取，这个时候会将misses加一，
	// 当misses累积到 dirty的长度的时候， 就会将dirty提升为read,避免从dirty中miss太多次。因为操作dirty需要加锁。
	misses int
}
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[interface{}]*entry
	// 如果Map.dirty有些数据不在m中，这个值为true
	amended bool
}

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	// *interface{}
	p unsafe.Pointer 
}
```
从源码可以看出，此锁保持了两个map，虽然会额外占据空间，但是并不大多少（典型空间换时间）。

##### 读数据流程
![](https://oscimg.oschina.net/oscnet/up-0bd60beac10ea70feb1e69eb9e5228bd47c.png)
##### 写数据流程
![](https://oscimg.oschina.net/oscnet/up-e7cd5f07a85183c9ddfe0e1667ae5f044b7.png)
##### 删数据流程
![](https://oscimg.oschina.net/oscnet/up-04e793394509a13c09bde98e4579be242f1.png)


##### 总结优化点

1. 无锁读与读写分离；
2. 写加锁与延迟提升；
3. 指针与惰性删除，map保存的value都是指针。惰性删除，实际删除是在 Store时候去check 然后删除。

| 实现方式  |  原理 |  适用场景 |
| ------------ | ------------ | ------------ |
|  map+Mutex | 通过Mutex互斥锁来实现多个goroutine对map的串行化访问	读写都需要通过Mutex加锁和释放锁 |  适用于读写比接近的场景 |
| map+RWMutex  | 通过RWMutex来实现对map的读写进行读写锁分离加锁，从而实现读的并发性能提高  | 同Mutex相比适用于读多写少的场景  |
| sync.Map  | 	底层通分离读写map和原子指令来实现读的近似无锁，并通过延迟更新的方式来保证读的无锁化	  | 读多修改少，元素增加删除频率不高的情况，在大多数情况下替代上述两种实现  |

