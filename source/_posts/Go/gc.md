---
title: Golang Garbage Collection
date: 2020-02-24 22:10
tags: golang
---
看了下runtime的代码（注释），总结一哈
<!--more-->

## 现在有的比较流行的GC

1. 简单的refcount，比如redis，即 引用多一个，refcount就+1

2. mark & sweep，标记然后用监视内存的程序或者lazy清理（这个一般不会），golang现在用的就是这种

3. 比较牛批的分代收集，如JAVA，什么新生代，老年代，eden等；不过这些都是

## Go的GC

### 步骤

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


#### 方法
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