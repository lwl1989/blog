CreateTime:2021-04-20 17:50:49.0

# 现在面试都这么直接的嘛？

面试难如狗，肝不过年轻人怎么办，只能多总结。

## slice

那么切片，就是今天的主角了。

直接搜哈。

### 问题1，slice的底层数据结构

我擦，这么直接的嘛？

我猜是数组加链表，结果猜错了0分。

翻看源码。

runtime/slice.go
```
type slice struct {
	array unsafe.Pointer  //数据结构是简单粗暴的数组指针
	len   int
	cap   int
}
```

### 问题2，slice是如何扩容的

又猜错了~

还是继续看源码吧

从源码找了半天，发现一个这。

	growslice handles slice growth during append.

so，就是你了。

```
func growslice(et *_type, old slice, cap int) slice { // 第三个cap,新的最小容量
	//巴拉巴拉巴拉 一堆判断

	newcap := old.cap  //变量存储空间大小
	doublecap := newcap + newcap  //双倍空间大小
	if cap > doublecap {  //如果历史空间大于双倍的容量,新的最小容量
		newcap = cap
	} else {
		//如果长度小于 1024  新长度就是2倍老容量
		if old.len < 1024 {
			newcap = doublecap
		} else {
			//当大于1024 走公式   newcap += newcap / 4,直到newcap大于等于老cap
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	//对et的size做匹配，获取需要申请的空间大小
	//1 不处理直接分配
	//系统指针大小 进行计算和位运算
	//2 唯一运算
	//默认 相乘
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	//如果append一次超过过多的元素新增，直接报错，越界，超出容量大小
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer //申请新的内存，并把指针指向p
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	//将老的数据移动到新的数据
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```


# 总结

其实可以看出，golang的切片扩容是比较粗暴的，直接赋值拷贝。不过，golang区分的长度和容量两种单位计量，一般会提前分配足够的cap,可以减少maclloc的次数。

