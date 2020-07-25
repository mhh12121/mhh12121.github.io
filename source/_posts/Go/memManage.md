---
title: Golang Memory Allocator
date: 2020-02-24 22:10
tags: golang
---

<!--more-->

# golang内存管理
go的内存管理是基于tcmalloc，[这个连接](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)看详情

任何大小的内存页可以被分割成**一系列同样大小的object**,这些规定的大小size则被定义在[sizetoclass](#sizetoclass),然后被一个**bitmap**管理

## 基本数据结构

类似于[TCMalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)

大概概括:
其目的是 减少多线程对内存请求时候的锁竞争，在对小内存的申请时甚至可以无锁操作，获取大内存时用spinlocks；但是其在TLS会预分配一部分空间，所以启动时相比dlmalloc等其他内存分配器空间较大，但是最终会接近;


**class_to_allocnpages**总共有67个范围

栈的分配也是多层次和多class的
- mspan (主要使用该机制减少碎片):

	被内存堆管理的的页面，至少一个页(8KB)，用于范围分配内存，比如16-32B则分配32B,112~128则分配128B的span

- mcentral 

	全局有 67 × 2 (?) 个对应不同size的span **后备**mcentral 
收集所有特定size的span，如果也被用完，则再次转向mheap申请

```go
type mheap struct{
	...
	// central free lists for small size classes.
	// the padding makes sure that the mcentrals are
	// spaced CacheLinePadSize bytes apart, so that each mcentral.lock
	// gets its own cache line.
	// central is indexed by spanClass.
	//numSpanClasses = _NumSizeClasses << 1 = 134
	central [numSpanClasses]struct {
		mcentral type mcentral struct {
					lock      mutex
					spanclass spanClass
					nonempty  mSpanList // list of spans with a free object, ie a nonempty free list
					empty     mSpanList // list of spans with no free objects (or cached in an mcache)

					// nmalloc is the cumulative count of objects allocated from
					// this mcentral, assuming all spans in mcaches are
					// fully-allocated. Written atomically, read under STW.
					nmalloc uint64
				}
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}
	....
}

```
代码中的pad是用作分割多个mcentral，以CacheLinePadSize个Bytes分割开，所以每一个mcentral的lock可以得到自己的cache line
我认为可以看做是内存对齐的一种方法(???是不是捏)

- mcache

	多层次的cache用来减少分配冲突，mcache是per-P的，所以无锁，mspan的每个P(process)下的可用cache空间；小于16B直接使用P中的macache
```go
	// Per-thread (in Go, per-P) cache for small objects.
// No locking needed because it is per-thread (per-P).
//
// mcaches are allocated from non-GC'd memory, so any heap pointers
// must be specially handled.
//
//因为只用作local P，所以自然无锁
//go:notinheap
//这个标志说明了不在heap中，也可以理解，per - P好明显不在公共heap中
type mcache struct {
	// The following members are accessed on every malloc,
	// so they are grouped here for better caching.
	next_sample uintptr // trigger heap sample after allocating this many bytes
	local_scan  uintptr // bytes of scannable heap allocated

	// Allocator cache for tiny objects w/o pointers.
	// See "Tiny allocator" comment in malloc.go.
	//具体可以见下面的tiny allocator分配器
	// tiny points to the beginning of the current tiny block, or
	// nil if there is no current tiny block.
	//
	// tiny is a heap pointer. Since mcache is in non-GC'd memory,
	// we handle it by clearing it in releaseAll during mark
	// termination.
	tiny             uintptr
	tinyoffset       uintptr
	local_tinyallocs uintptr // number of tiny allocs not counted in other stats

	// The rest is not accessed on every malloc.

	alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass

	stackcache [_NumStackOrders]stackfreelist

	// Local allocator stats, flushed during GC.
	local_largefree  uintptr                  // bytes freed for large objects (>maxsmallsize)
	local_nlargefree uintptr                  // number of frees for large objects (>maxsmallsize)
	local_nsmallfree [_NumSizeClasses]uintptr // number of frees for small objects (<=maxsmallsize)

	// flushGen indicates the sweepgen during which this mcache
	// was last flushed. If flushGen != mheap_.sweepgen, the spans
	// in this mcache are stale and need to the flushed so they
	// can be swept. This is done in acquirep.
	flushGen uint32
}
```

- fixalloc

一个不定长度的列表，用来管理**不在堆上**的固定的对象，这些对象都是runtime上大小固定的结构，比如mspan，mcache

- mheap:

	全局只有一个
内存堆，以页为粒度(8KB)进行管理,结构体为treap，维护空闲连续page，归还内存到heap中时，连续地址会合并；大于32KB内存申请直接从mheap中拿，剩下的则先使用当前P的mcache中对应的size class分配，如果其对应的span已经无可用的块，则向mcentral请求，如果没有则在mheap申请，如果还不够则要向操作系统申请;

- mstatisitc

	提供管理信息


### mspan的结构体

```go
type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.

	startAddr uintptr // address of first byte of span aka s.base()
	npages    uintptr // number of pages in span

	manualFreeList gclinkptr // list of free objects in mSpanManual spans

	// freeindex is the slot index between 0 and nelems at which to begin scanning
	// for the next free object in this span.
	// Each allocation scans allocBits starting at freeindex until it encounters a 0
	// indicating a free object. freeindex is then adjusted so that subsequent scans begin
	// just past the newly discovered free object.
	//
	// If freeindex == nelem, this span has no free objects.
	//
	// allocBits is a bitmap of objects in this span.
	// If n >= freeindex and allocBits[n/8] & (1<<(n%8)) is 0
	// then object n is free;
	// otherwise, object n is allocated. Bits starting at nelem are
	// undefined and should never be referenced.
	//
	// Object n starts at address n*elemsize + (start << pageShift).
	freeindex uintptr
	// TODO: Look up nelems from sizeclass and remove this field if it
	// helps performance.
	nelems uintptr // number of object in the span.

	// Cache of the allocBits at freeindex. allocCache is shifted
	// such that the lowest bit corresponds to the bit freeindex.
	// allocCache holds the complement of allocBits, thus allowing
	// ctz (count trailing zero) to use it directly.
	// allocCache may contain bits beyond s.nelems; the caller must ignore
	// these.
	allocCache uint64

	// allocBits and gcmarkBits hold pointers to a span's mark and
	// allocation bits. The pointers are 8 byte aligned.
	// There are three arenas where this data is held.
	// free: Dirty arenas that are no longer accessed
	//       and can be reused.
	// next: Holds information to be used in the next GC cycle.
	// current: Information being used during this GC cycle.
	// previous: Information being used during the last GC cycle.
	// A new GC cycle starts with the call to finishsweep_m.
	// finishsweep_m moves the previous arena to the free arena,
	// the current arena to the previous arena, and
	// the next arena to the current arena.
	// The next arena is populated as the spans request
	// memory to hold gcmarkBits for the next GC cycle as well
	// as allocBits for newly allocated spans.
	//
	// The pointer arithmetic is done "by hand" instead of using
	// arrays to avoid bounds checks along critical performance
	// paths.
	// The sweep will free the old allocBits and set allocBits to the
	// gcmarkBits. The gcmarkBits are replaced with a fresh zeroed
	// out memory.
	allocBits  *gcBits
	gcmarkBits *gcBits

	// sweep generation:
	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
	// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
	// h->sweepgen is incremented by 2 after every GC

	sweepgen    uint32
	divMul      uint16     // for divide by elemsize - divMagic.mul
	baseMask    uint16     // if non-0, elemsize is a power of 2, & this will get object allocation base
	allocCount  uint16     // number of allocated objects
	spanclass   spanClass  // size class and noscan (uint8)
	state       mSpanState // mspaninuse etc
	needzero    uint8      // needs to be zeroed before allocation
	divShift    uint8      // for divide by elemsize - divMagic.shift
	divShift2   uint8      // for divide by elemsize - divMagic.shift2
	scavenged   bool       // whether this span has had its pages released to the OS
	elemsize    uintptr    // computed from sizeclass or from npages
	limit       uintptr    // end of data in span
	speciallock mutex      // guards specials list
	specials    *special   // linked list of special records sorted by offset.
}

```

主要的字段:

- elementsize: slot大小，B为单位
- freeindex，<该值的已经被分配，>=该位置的可能未被分配，需要配合allocCache查找,每次分配后，freeindex设置为分配的slot+1
- allocBits表示上一次GC之后哪一些slot被使用，0未使用或释放，1已分配
- allocCache表示从freeindex开始的64个slot的分配情况，1为未分配，0为已分配，使用ctz(Count trailing zeros指令)找到第一个非0位，使用完了就从allocBits加载，取反；

- 每次gc完的sweep阶段，将allocBits设置为gcmarkbits


## 内存总体结构

暂时将linux amd64作为例子

- 1.10以前，内存不是初始化就分配虚拟内存
arena大小为512G，为了方便将其分为一个个page，所以总共也有512G/8KB = 65536个page

	span区域存放指向span的指针，表示arena区域page所属的span，所以其大小即为 512GB/8KB* 8B(指针大小) = 512M 

	bitmap主要用于GC，两个bit表示arena中一个字的可用状态，所以表示为 (512GB/ 8(8个byte一个字，即指令长度)) * 2 /8 (8个bit一个byte) = 16G 长度

- 1.11以后
	改成两阶段稀疏索引方式，内存允许超过512G，也可以允许不连续内存
	mheap中的arenas字段实际是一个指针数组，每个heapArena管理一个64MB的内存
bitmap和spans功能不变

## go的分配内存策略：

### 小对象（小于32K）和大对象的不同
1. 小于32KB的小对象时，会直接去 P (process)的cache的list里面拿，大于32KB的才去堆拿;
拿的过程会roundup一下大小，然后在当前P的mcachexmspan

```golang
// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
//typ有两种，一种noscan，一种是scan，表示分配对象是否包含指针
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if gcphase == _GCmarktermination {
		throw("mallocgc called with gcphase == _GCmarktermination")
	}

	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}
	......
	
	// assistG is the G to charge for this allocation, or nil if
	// GC is not currently active.
	var assistG *g
	if gcBlackenEnabled != 0 {
		// Charge the current user G for this allocation.
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		// Charge the allocation against the G. We'll account
		// for internal fragmentation at the end of mallocgc.
		assistG.gcAssistBytes -= int64(size)

		if assistG.gcAssistBytes < 0 {
			// This G is in debt. Assist the GC to correct
			// this before allocating. This must happen
			// before disabling preemption.
			gcAssistAlloc(assistG)
		}
	}

	// Set mp.mallocing to keep from being preempted by GC.
	mp := acquirem()
	if mp.mallocing != 0 {
		throw("malloc deadlock")
	}
	if mp.gsignal == getg() {
		throw("malloc during signal")
	}
	mp.mallocing = 1

	shouldhelpgc := false
	dataSize := size
	c := gomcache()
	var x unsafe.Pointer

	noscan := typ == nil || typ.ptrdata == 0
	if size <= maxSmallSize {
		...
		//tiny allocator分配方式的代码
	} else {
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			s = largeAlloc(size, needzero, noscan)
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}

	var scanSize uintptr
	if !noscan {
		// If allocating a defer+arg block, now that we've picked a malloc size
		// large enough to hold everything, cut the "asked for" size down to
		// just the defer header, so that the GC bitmap will record the arg block
		// as containing nothing at all (as if it were unused space at the end of
		// a malloc block caused by size rounding).
		// The defer arg areas are scanned as part of scanstack.
		if typ == deferType {
			dataSize = unsafe.Sizeof(_defer{})
		}
		heapBitsSetType(uintptr(x), size, dataSize, typ)
		if dataSize > typ.size {
			// Array allocation. If there are any
			// pointers, GC has to scan to the last
			// element.
			if typ.ptrdata != 0 {
				scanSize = dataSize - typ.size + typ.ptrdata
			}
		} else {
			scanSize = typ.ptrdata
		}
		c.local_scan += scanSize
	}

	// Ensure that the stores above that initialize x to
	// type-safe memory and set the heap bits occur before
	// the caller can make x observable to the garbage
	// collector. Otherwise, on weakly ordered machines,
	// the garbage collector could follow a pointer to x,
	// but see uninitialized memory or stale heap bits.
	publicationBarrier()

	// Allocate black during GC.
	// All slots hold nil so no scanning is needed.
	// This may be racing with GC so do it atomically if there can be
	// a race marking the bit.
	if gcphase != _GCoff {
		gcmarknewobject(uintptr(x), size, scanSize)
	}

	if raceenabled {
		racemalloc(x, size)
	}

	if msanenabled {
		msanmalloc(x, size)
	}

	mp.mallocing = 0
	releasem(mp)

	if debug.allocfreetrace != 0 {
		tracealloc(x, size, typ)
	}

	if rate := MemProfileRate; rate > 0 {
		if rate != 1 && size < c.next_sample {
			c.next_sample -= size
		} else {
			mp := acquirem()
			profilealloc(mp, x, size)
			releasem(mp)
		}
	}

	if assistG != nil {
		// Account for internal fragmentation in the assist
		// debt now that we know it.
		assistG.gcAssistBytes -= int64(size - dataSize)
	}

	if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}

	return x
}
```


### 各种分配器

#### TinyAllocator
上面的源代码中，当size<=maxSmallSize时，实际上调用了一种tinyAllocator的分配器

该分配是 **会结合多个申请内存请求 成为 申请一个内存块的请求(即合并小对象存储下来)**；
所以当所有的子对象都不可达时，这个内存块才会被释放（同时这些对象一定不含指针，这个很好理解，有指针又要从指针处进行可达性查询）
作用是**避免可能内存浪费**

其中这个maxTinySize是可调整的，调整成8bytes就一定不会浪费任何内存（一个页就8B），但是这样就没什么意义（不用combine）
照这个道理，16bytes就可能会造成2× 最坏情况的内存浪费，32B就会造成4×最坏情况的内存浪费

- 其中从tinyallocator分配的对象不能被显式释放(?)，所以每次我们显式释放对象的时候，自然的要保证这个对象是大于maxTinySize

- 主要的对象是一些小字符串和一些独立的变量，json的benchmark使用了这个分配器，减少了20%的堆size

其结构![如图](/img/tinyObjects.png)

- 每个P在本地维护了专门的memory block来存储tinyObject，分配时根据tinyoffset和需要的size及对齐来判断该block是否容纳该object，如果可以就返回地址

```go

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	....
	//maxSmallsize = 32786
	if size<=maxSmallSize {
		//申请对象不含指针且小于16B则会调用tiny allocator
		if noscan && size < maxTinySize {
			// Tiny allocator.
			//
			//一种分配器
			// Tiny allocator combines several tiny allocation requests
			// into a single memory block. The resulting memory block
			// is freed when all subobjects are unreachable. The subobjects
			// must be noscan (don't have pointers), this ensures that
			// the amount of potentially wasted memory is bounded.
			//
			// Size of the memory block used for combining (maxTinySize) is tunable.
			// Current setting is 16 bytes, which relates to 2x worst case memory
			// wastage (when all but one subobjects are unreachable).
			// 8 bytes would result in no wastage at all, but provides less
			// opportunities for combining.
			// 32 bytes provides more opportunities for combining,
			// but can lead to 4x worst case wastage.
			// The best case winning is 8x regardless of block size.
			//
			// Objects obtained from tiny allocator must not be freed explicitly.
			// So when an object will be freed explicitly, we ensure that
			// its size >= maxTinySize.
			//
			// SetFinalizer has a special case for objects potentially coming
			// from tiny allocator, it such case it allows to set finalizers
			// for an inner byte of a memory block.
			//
			// The main targets of tiny allocator are small strings and
			// standalone escaping variables. On a json benchmark
			// the allocator reduces number of allocations by ~12% and
			// reduces heap size by ~20%.
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				//该对象适合于已有的tinyblock
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			span := c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			//在32KB以内，16B以上的
			var sizeclass uint8
			//smallSizeMax = 1024
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span := c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	}else{
		....
	}
	....
}
```

#### fixAlloc

因为我们的都知道go分配对象是在go gc heap中，并且由mspan，mcache，mcentral这些结构管理，但是这些结构的对象又是在哪里管理和分配呢？

fixalloc就是做这个的：
前面讲到fixalloc都是mheap中固定的结构

- 主要目的就是一次性分配一大块内存(注意persistentalloc方法，使用是mmap，不指定地址，分配内存不再arena范围内，从进程空间获得可能百来KB)，
每次请求对应的结构体大小，释放时就放在list链表中

大概的分配有以下集中
```go
type mheap struct{
	...
	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	treapalloc            fixalloc // allocator for treapNodes*
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
	arenaHintAlloc        fixalloc // allocator for arenaHints
	...
}
```

#### stackCache

在mcache结构上

```go
// Number of orders that get caching. Order 0 is FixedStack
	// and each successive order is twice as large.
	// We want to cache 2KB, 4KB, 8KB, and 16KB stacks. Larger stacks
	// will be allocated directly.
	// Since FixedStack is different on different systems, we
	// must vary NumStackOrders to keep the same maximum cached size.
	//   OS               | FixedStack | NumStackOrders
	//   -----------------+------------+---------------
	//   linux/darwin/bsd | 2KB        | 4
	//   windows/32       | 4KB        | 3
	//   windows/64       | 8KB        | 2
	//   plan9            | 4KB        | 3
_NumStackOrders = 4 - sys.PtrSize/4*sys.GoosWindows - 1*sys.GoosPlan9
type mcache struct{
	...
	stackcache [_NumStackOrders]stackfreelist 
	...
}
type stackfreelist struct {
	list gclinkptr // linked list of free stacks
	size uintptr   // total size of stacks in list
}
```
大概结构![如图](/img/stackCache.png)

stackCache是per-P的，在另外一篇文章[goroutine](../goroutine.html)上讲过，主要用于分配goroutine的stack，同普通内存一样
其分为多个segment，class, linux就分为2KB,4KB,8KB,16KB等级

其中 > 16K的直接从全局stacklarge分配
否则按照先从P的stackcache分配=> 如果无法分配 => 从全局stackpool分配一批stack(stackpoolalloc)，赋给该p的stackcache，再从local stackcache分配

### 一些重要参数

- go_memstats_sys_bytes: 进程从操作系统获得内存的总字节数，包含了go运行的stack，heap还有其他数据结构相关的虚拟地址空间

- go_memstats_heap_inuse_bytes: 在span中真正被使用的字节数；其中不包括可能已经返回到操作系统，或者可以重用进行对分配、可以将作为堆栈内存重用的字节 (?)

- go_memstats_heap_idle_bytes: 在span中空闲的字节数;

- go_memstats_stack_sys_bytes: 栈内存字节数；主要用于goroutine栈内存的分配;


由以上参数结合代码其实可以知道大概span在内存中有几种状态:

1. idle不包含对象或者其他数据，空闲的物理内存可以释放回OS（虚拟地址不会释放！！！），或者将其转换成inuse状态或者stack span

2. inuse,至少包含一个mheap，并且可能有空闲空间分配更多堆对象

3. stack span，只会在堆或者是栈内存其中之一



## 内存对齐以及一些分配规则
runtime/msize.go
```golang
func roundupsize(size uintptr) uintptr {
	//size<32768
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 {
			//这里面的字段是go对特定class设定的对应大小
			return uintptr(class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]])
		} else {
			return uintptr(class_to_size[size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]])
		}
	}
	//size为负数,_PageSize=1<<13 
	if size+_PageSize < size {
		return size
	}
	return round(size, _PageSize)
}
//该运算在下面会提到
// round n up to a multiple of a.  a must be a power of 2.
func round(n, a uintptr) uintptr {
	return (n + a - 1) &^ (a - 1)
}
```

注意到**class_to_size**和**size_to_class**等等字段

<a id="sizetoclass">[sizetoclass]</a>
实际上在runtime/sizeclasses.go里面可以体现出go对不同大小的class设置的size：
每个span都带有一个sizeclass，即表明该span的page应该被怎么用；


>> class0表示单独分配一个>32KB对象的span，有67个size，每个size有两种，分配用于有指针和无指针对象，所以有个67*2	= 134个class (即上面提到的numSpanClasses)


```go
// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         32        8192      256           0     46.88%
//     4         48        8192      170          32     31.52%
//     5         64        8192      128           0     23.44%
//     6         80        8192      102          32     19.07%
//     7         96        8192       85          32     15.95%
//     8        112        8192       73          16     13.56%
//     9        128        8192       64           0     11.72%
//    10        144        8192       56         128     11.82%
//    11        160        8192       51          32      9.73%
......
//    60      19072       57344        3         128      3.57%
//    61      20480       40960        2           0      6.87%
//    62      21760       65536        3         256      6.25%
//    63      24576       24576        1           0     11.45%
//    64      27264       81920        3         128     10.00%
//    65      28672       57344        2           0      4.91%
//    66      32768       32768        1           0     12.50%
```
可以看到bytes/obj一栏，就是go预定义object大小，最小是8B，最大是32KB（注意这里只是在32KB以内，还有大于32KB以外的），所以都可以解释到**slice**在扩容的时候可能会不遵守*2和1.25倍扩容的规则；

相关的main方法可以在classToSize的转换runtime/mksizeclass.go中找到



###  golang的位运算
有时候会经常看见会用一些全局常量：
```go
//指针大小,一般64位就是8
const PtrSize = 4 << (^uintptr(0) >> 63) //8
```
sys.PtrSize, sys.RegSize等等
#### 1. ^运算

- 用作单目运算时， ^ 指的就是取反,等于一些语言的 ~ 符号（这里注意都一样取补码）

ps: 这里复习一下，


正数取反：化为二进制，得到补码(正数补码和原码一样)，再对补码每位取反

负数取反：化为二进制，得到补码(所有除符号位的每位取反，+1)，然后再对补码全部每位取反
```go
x:=^3
//3=》 0011=》 1100=-4 
log.Printf("%d",x)//-4

x:=^(-3)
//-3=》 1011 =》 1100 =》 1101 =》 0010=2
log.Printf("%d",x)//2

```
也可以用比较直接的方法：
^a= -(a+1)
- 用作双目运算符时则为异或（XOR）
    相同为0，相异为1


#### 2. &^运算

将运算符号左边数据相异保留，相同置为0;

符合：
- 右侧为0，左侧数不变，
- 右侧是1，左侧清零
- 符合结合法即 a&^b=a&(^b)

经常用该符号作内存对齐
如runtime/stubs.go里面
```go
// round n up to a multiple of a.  a must be a power of 2.
func round(n, a uintptr) uintptr {
	return (n + a - 1) &^ (a - 1)
}
//可以有这种说法：
//找到最大位为１的位数，然后用１左移该位数即是roundup后的结果,
//比如6 : 110,最大为为1的是在第三位，1<<3 = 1000 = 8,即十进制的8
// n=6,a=2 : 110 => (6+2-1) = 111 &^ 001 = 110  
```
runtime/malloc.go