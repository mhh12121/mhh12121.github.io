---
title: Golang Garbage Collection
date: 2020-02-24 22:10
tags: golang
---
看了下runtime的<s>代码</s>（注释），总结一哈

<!--more-->
## 目的

对于所有的垃圾回收器的目的（指标）无非就是以下几个：

1. 所占用程序的时间，停顿时间
2. 频率
3. CPU占比
4. 内存占比（堆开销）
5. 内存的分配方式（碎片化程度）
6. 内存释放方式
7. 并发效果
8. 是否智能化（根据某些系统条件进行调节）
9. 是否可以自定义化参数 

## 静态语言类一般会在程序的三个阶段涉及gc的操作

- 编译期
- 运行时内存分配
- 运行时扫描

## 现在有的比较流行的GC

1. 简单的refcount，比如redis，即 引用多一个，refcount就+1

2. mark & sweep，标记然后用监视内存的程序或者lazy清理（这个一般不会），golang现在用的就是这种

3. 比较牛批的分代收集，如JAVA，什么新生代，老年代，eden等；不过这些都是比较老的比如JAVA一类的

## Go的GC

### 总流程
- 清理终止阶段；
	暂停程序，所有的处理器在这时会进入安全点（Safe point）；
	如果当前垃圾收集循环是强制触发的，我们还需要处理还未被清理的内存管理单元；
- 标记阶段；
	将状态切换至 `_GCmark`、开启写屏障、用户程序协助（Mutator Assiste）并将根对象入队；
	恢复执行程序，标记进程和用于协助的用户程序会开始并发标记内存中的对象，写屏障会将被覆盖的指针和新指针都标记成灰色，而所有新创建的对象都会被直接标记成黑色；
	开始扫描根对象，包括所有 Goroutine 的栈、全局对象以及不在堆中的运行时数据结构，扫描 Goroutine 栈期间会暂停当前处理器；
	依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色；
	使用分布式的终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段；
- 标记终止阶段；
	暂停程序、将状态切换至 _GCmarktermination 并关闭辅助标记的用户程序；
	清理处理器上的线程缓存；
- 清理阶段；
	将状态切换至 _GCoff 开始清理阶段，初始化清理状态并关闭写屏障；
	恢复用户程序，所有新创建的对象会标记成白色；
	后台并发清理所有的内存管理单元，当 Goroutine 申请新的内存管理单元时就会触发清理；

### 1. 编译阶段

- 内存对齐
略，这个可以参考自己的memManage

- 初始化一些字段
我们直接
```go
type _type struct {
	...
	ptrdata    uintptr // size of memory prefix holding all pointers
	...
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	
	...
}
```


举个例子
```go
type testStruct struct{
	ptr uintptr//8
	A uint8 //1
	B *uint8//8
	C uint32//4
	D *uint64//8
	E uint64//8
}
```
我直接打点在`mbitmap.go:947`即`heapBitSetType`上面

```yml
*runtime._type {
	size: 376, //该对象有多少个字(64位一个字=64bits=8Bytes)
	ptrdata: 360, //其中的指针有多少个字
	hash: 3901217204, 
	tflag: tflagUncommon|tflagExtraStar|tflagNamed (7),
	align: 8, fieldAlign: 8, 
	kind: 25,
	equal: nil, 
	gcdata: *112, //0111 0000
	str: 11827, 
	ptrToThis: 41536}

```

- ptrdata

	指针截止的长度


- 重点看这个字段 `gcdata` :

	`112`的二进制就是 `0111 0000`, 而`0111 0000` reverse一下就变成`(0000 1110)`<sub>2</sub> ，第2,3,4个bit为1，分别对应


### 2. 运行阶段

主要是`heapBitsSetType`这个函数

```go
func heapBitsSetType(x, size, dataSize uintptr, typ *_type) {}
```
其传入参数可以看到:
- x :

结构(对象)的**开始地址**，是uintptr

- size: 



- dataSize: 
	
	其值永远都是每次roundup（可能根据**sizetoclass(可以看下memManage那篇文章)**)得到的大小,
	但是，在分配**defer块**的时候**不会**,sizeof(defer{})可以看见至少有6个字(6*64bits=48 Bytes,64位下),可能会偏大;

- typ: 

传入的类型，记录了结构的gc的map(gcdata)，大小，类型，hash等一系列值


为什么不用原子性保证，并发问题？注释里面给出答案：
因为每次都
- 只会从一个span里面分配空间
- span的bitmap每次都只会在规定的byte边界内（内存对齐的结果）

所以写的冲突是不会出现的；


```go

// There can only be one allocation from a given span active at a time,
// and the bitmap for a span always falls on byte boundaries,
// so there are no write-write races for access to the heap bitmap.
// Hence, heapBitsSetType can access the bitmap without atomics.
//
```

主要逻辑:

```go
func heapBitsSetType(x, size, dataSize uintptr, typ *_type) {
	...
	//1. 通过分配地址反查到heap的heapBits结构
	h := heapBitsForAddr(x)
	//获取到类型的指针bitmap
	ptrmask := typ.gcdata // start of 1-bit pointer mask (or GC program, handled below)
	...
	var ( ...)
	//将h.bitp堆上的bitmap取出
	hbitp = h.bitp
	...
	//该类型的bitmap
	p = ptrmask

	....


	if p != nil {
		//保存bitmap第一个Byte
		b = uintptr(*p)
		//p指向下一个Byte
		p = add1(p)
		nb = 8
	}
	//我们的结构是48==48，最简单的struct
	if typ.size == dataSize {
		// Single entry: can stop once we reach the non-pointer data.
		//nw = 5 = 40 / 8  ，说明扫描到第5个字段即可,因为ptrdata就是已经划定了范围[0,40]
		nw = typ.ptrdata / sys.PtrSize
	} else {
		//针对array的
		// Repeated instances of typ in an array.
		// Have to process first N-1 entries in full, but can stop
		// once we reach the non-pointer data in the final entry.
		nw = ((dataSize/typ.size-1)*typ.size + typ.ptrdata) / sys.PtrSize
	}
	//如果nw=0，该struct无指针
	if nw == 0 {
		// No pointers! Caller was supposed to check.
		println("runtime: invalid type ", typ.string())
		throw("heapBitsSetType: called with non-pointer type")
		return
	}
	//至少要写入两个字，因为noscan 的编码要求todo???
	if nw < 2 {
		// Must write at least 2 words, because the "no scan"
		// encoding doesn't take effect until the third word.
		nw = 2
	}

	//接下来较为重要:
	// Phase 1: Special case for leading byte (shift==0) or half-byte (shift==2).
	// The leading byte is special because it contains the bits for word 1,
	// which does not have the scan bit set.
	// The leading half-byte is special because it's a half a byte,
	// so we have to be careful with the bits already there.
	switch {
	default:
		throw("heapBitsSetType: unexpected shift")

	case h.shift == 0:
		// Ptrmask and heap bitmap are aligned.
		// Handle first byte of bitmap specially.
		//
		// The first byte we write out covers the first four
		// words of the object. The scan/dead bit on the first
		// word must be set to scan since there are pointers
		// somewhere in the object. The scan/dead bit on the
		// second word is the checkmark, so we don't set it.
		// In all following words, we set the scan/dead
		// appropriately to indicate that the object contains
		// to the next 2-bit entry in the bitmap.
		//
		// TODO: It doesn't matter if we set the checkmark, so
		// maybe this case isn't needed any more.
		//b是类型的,b = 0001 0100
		//bitPointerAll = 0000 1111
		//hb = 0000 0100
		hb = b & bitPointerAll
		
		hb |= bitScan | bitScan<<(2*heapBitsShift) | bitScan<<(3*heapBitsShift)
		if w += 4; w >= nw {
			goto Phase3
		}
		
		*hbitp = uint8(hb)
		//指针往后一个字节(递进一个)
		hbitp = add1(hbitp)
		b >>= 4
		nb -= 4

	case sys.PtrSize == 8 && h.shift == 2:
		...
	}

	
	// Phase 2: Full bytes in bitmap, up to but not including write to last byte (full or partial) in bitmap.
	// The loop computes the bits for that last write but does not execute the write;
	// it leaves the bits in hb for processing by phase 3.
	// To avoid repeated adjustment of nb, we subtract out the 4 bits we're going to
	// use in the first half of the loop right now, and then we only adjust nb explicitly
	// if the 8 bits used by each iteration isn't balanced by 8 bits loaded mid-loop.
	//继续处理后4个bit!!!
	nb -= 4
	for {
		// Emit bitmap byte.
		// b has at least nb+4 bits, with one exception:
		// if w+4 >= nw, then b has only nw-w bits,
		// but we'll stop at the break and then truncate
		// appropriately in Phase 3.
		hb = b & bitPointerAll
		hb |= bitScanAll
		if w += 4; w >= nw {
			//已经处理完成，有指针的字段都包含在已经处理的ptrmask范围内
			break
		}
		...
	}

//第三阶段,写入最后的byte或者是部分byte然后将剩下的bitmap置0
Phase3:
	// Phase 3: Write last byte or partial byte and zero the rest of the bitmap entries.
	if w > nw {
		// Counting the 4 entries in hb not yet written to memory,
		// there are more entries than possible pointer slots.
		// Discard the excess entries (can't be more than 3).
		mask := uintptr(1)<<(4-(w-nw)) - 1
		hb &= mask | mask<<4 // apply mask to both pointer bits and scan bits
	}

	// Change nw from counting possibly-pointer words to total words in allocation.
	nw = size / sys.PtrSize

	// Write whole bitmap bytes.
	// The first is hb, the rest are zero.
	if w <= nw {
		*hbitp = uint8(hb)
		hbitp = add1(hbitp)
		hb = 0 // for possible final half-byte below
		for w += 4; w <= nw; w += 4 {
			*hbitp = 0
			hbitp = add1(hbitp)
		}
	}

	// Write final partial bitmap byte if any.
	// We know w > nw, or else we'd still be in the loop above.
	// It can be bigger only due to the 4 entries in hb that it counts.
	// If w == nw+4 then there's nothing left to do: we wrote all nw entries
	// and can discard the 4 sitting in hb.
	// But if w == nw+2, we need to write first two in hb.
	// The byte is shared with the next object, so be careful with
	// existing bits.
	if w == nw+2 {
		*hbitp = *hbitp&^(bitPointer|bitScan|(bitPointer|bitScan)<<heapBitsShift) | uint8(hb)
	}


Phase4:
	// Phase 4: Copy unrolled bitmap to per-arena bitmaps, if necessary.
	...
}
```


`heapSetBitsType`函数实际上主要做的就是 设置 `h.bitp`这个值,
每分配一块内存，都会有一个bitmap对应这个内存块，指明指针的位置


### 运行扫描阶段

主要由两个行为：
#### 1. scanstack
从markroot开始，栈 、全局变量、寄存器等根对象开始扫描，
创建一个DAG，将root对象放入一个队列中；

```go


```


#### 2. scanobject
异步的goroutine运行`gcDrain`函数，从队列里消费对象，




### 一些gc相关的设置

#### 触发
触发gc可以由runtime.GC()(不一定,会判定是否执行)

提供了两个参数来控制GC 
#### 1.GCPercent
>> The first one is GCPercent. Basically this is a knob that adjusts how much CPU you want to use and how much memory you want to use. The default is 100 which means that half the heap is dedicated to live memory and half the heap is dedicated to allocation. You can modify this in either direction.

代码在runtime/mgc.go中:
```go
// Initialized from $GOGC.  GOGC=off means no GC.
var gcpercent int32
......
//go:linkname setGCPercent runtime/debug.setGCPercent
func setGCPercent(in int32) (out int32) {
	// Run on the system stack since we grab the heap lock.
	systemstack(func() {
		lock(&mheap_.lock)
		out = gcpercent
		if in < 0 {
			in = -1
		}
		gcpercent = in
		heapminimum = defaultHeapMinimum * uint64(gcpercent) / 100
		// Update pacing in response to gcpercent change.
		gcSetTriggerRatio(memstats.triggerRatio)
		unlock(&mheap_.lock)
	})
	// Pacing changed, so the scavenger should be awoken.
	wakeScavenger()

	// If we just disabled GC, wait for any concurrent GC mark to
	// finish so we always return with no GC running.
	if in < 0 {
		gcWaitOnMark(atomic.Load(&work.cycles))
	}

	return out
}

```
- 默认值是100，意味着一半的堆会用于实时内存，另一半的堆会用来分配
- 

#### 2.Maxheap
>> MaxHeap, which is not yet released but is being used and evaluated internally, lets the programmer set what the maximum heap size should be. Out of memory, OOMs, are tough on Go; temporary spikes in memory usage should be handled by increasing CPU costs, not by aborting. Basically if the GC sees memory pressure it informs the application that it should shed load. Once things are back to normal the GC informs the application that it can go back to its regular load. MaxHeap also provides a lot more flexibility in scheduling. Instead of always being paranoid about how much memory is available the runtime can size the heap up to the MaxHeap.

- 还在实验中... 最主要提供一些监控，会提示程序内存不足；还会使调度更加灵活；


#### 方法-三色回收
golang现在使用的是叫 **三色回收**的东西：
比较旧的版本:
大概步骤：

1. 所有对象初始都设为 **白色**
2. 从RootSet出发（即堆里面的对象，比如全局变量，所有的栈对象等），标记第一次所有可达到的对象为**灰色** ,这个过程,这里会stop the world;
3. 然后紧接着在第一个发现的各个对象上继续寻找引用这些对象的对象们，找到后（或者在这些基础上已经找不到了）就把第一次所有可达的对象转为**黑色**,这时会start the world;
而这些找到的对象们就标为**灰色** ;
3. 重复第二,三步,直到所有

```golang


```

#### 读写屏障

现在采用Dijkstra插入写屏障和Yuusa删除写屏障构成了混合写屏障:

主要用处是将 被覆盖的对象标记成**灰色** 并 在**当前栈没有扫描时**将新对象也标记成灰色:

伪代码:
```
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

**为了移除栈的重扫描过程**，除了引入混合写屏障之外,
在垃圾收集的`标记阶段`，我们还需要将创建的所有新对象都标记成**黑色**，防止新分配的栈内存和堆内存中的对象被错误地回收，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间;

### 做过的一些优化

#### size segregated span （分片隔离）
1. garbage collector要快速地找到object的开始位置，如果能知道在某个span中的object的大小，就可以直接往下舍入查找到位置
2. 更小的碎片化
3. 内部结构化？
4. 

### 调查一下其他部分
有这个一个变量： **gcphase**
```go
// Garbage collector phase.
// Indicates to write barrier and synchronization task to perform.
var gcphase uint32
```
write barrier写屏障
```go
// The compiler knows about this variable.
// If you change it, you must change builtin/runtime.go, too.
// If you change the first four bytes, you must also change the write
// barrier insertion code.
var writeBarrier struct {
	enabled bool    // compiler emits a check of this before calling write barrier
	pad     [3]byte // compiler uses 32-bit load for "enabled" field
	needed  bool    // whether we need a write barrier for current GC phase
	cgo     bool    // whether we need a write barrier for a cgo check
	alignme uint64  // guarantee alignment so that compiler can use a 32 or 64-bit load
}
```




### 具体实现

1. bitmap
/runtime/mbitmap.go
使用bitmap