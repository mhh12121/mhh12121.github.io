---
title: Golang Sync Package
date: 2020-10-02 15:01
tags: golang
---


# Sync包的一些笔记

<!--more-->

## Sync.Once


```go
// Once is an object that will perform exactly one action.
type Once struct {
	// done indicates whether the action has been performed.
	// It is first in the struct because it is used in the hot path.
	// The hot path is inlined at every call site.
	// Placing done first allows more compact instructions on some architectures (amd64/x86),
	// and fewer instructions (to calculate offset) on other architectures.
	done uint32
	m    Mutex
}
```

`once.Do()`方法中注意下要进行一次原子操作和一次加锁

```go
// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
// 	var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// 	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
    //
    //注意，上面是不正确的，因为要保证，atomic.StoreUint32在f返回后才能调用;
    //两个goroutine同时进来call f，快的一个会直接call f，这个时候慢的那一个不会等待快的f完成，而是直接返回
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
    defer o.m.Unlock()
     //这里需要再次重新判断下，因为atomic.LoadUint32取出状态值到o.m.Lock之间是有可能存在其它gotoutine改变status的状态值的
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```


## sync.WaitGroup

首先看下其结构：
旧的结构
```go
type WaitGroup struct {
	noCopy noCopy 
	state1 [12]byte//对齐有4个byte要浪费空间
	sema   uint32
}
```
现在的结构
```go
// A WaitGroup waits for a collection of goroutines to finish.
// The main goroutine calls Add to set the number of
// goroutines to wait for. Then each of the goroutines
// runs and calls Done when finished. At the same time,
// Wait can be used to block until all goroutines have finished.
//
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	//64位: (goroutine计数器)(等待计数器)(信号量)
	//不是64位的组成: (信号量)(goroutine计数器)(等待计数器)
	state1 [3]uint32
}
```

- 带有nocopy，即使用时只可以传指针
	copy的隐藏含义是，这个结构体在传参的时候所有字段都会被复制，那么试想一下多线程下，同时对`state1`进行修改，明显不能保证线程安全;

- state1是一个3个uint32元素的数组，高32位为计数器（当前仍未执行结束的goroutine数目），第二个32位为等待goroutine完成的数目（有多少个等待者）,
第三个可以理解为信号量semaphore， 至于为什么是3个元素，这个跟内存对齐有关(以前的结构不一样会浪费4个byte)，如下:


```go
// state returns pointers to the state and sema fields stored within wg.state1.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	//判断是否8位对齐
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		//前面8个bytes做uint64指针statep，后面4bytes做sema
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		//不是64位的组成: (信号量)(goroutine计数器)(等待计数器)
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}

```

`Add(delta)`方法: 将delta加到`[0]state1` 中
`Wait()`,[1]state1(waiter counter) 减一;
`Done()`，[0]state1(counter)减一;

## Sync.Mutex


互斥锁, 分为正常模式和饥饿模式
结构体，注意也是`nocopy`的:

```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}
```
`sema`就是Semaphore(信号量，mutex实际就是一种特殊的信号量)，平常用的`runtime.Semacquire`、`runtime.SemaRelease`等函数使用的用作同步的原语；


其中`state`有几个状态:

- `mutextLocked`
- `mutexWoken`
- `mutexStarving`
- `mutexWaiterShift`

`starvationThresholdNs`:饥饿状态下,为了保证公平性，占用的过期时间

```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	// Mutex fairness.
	//
	// Mutex can be in 2 modes of operations: normal and starvation.
	// In normal mode waiters are queued in FIFO order, but a woken up waiter
	// does not own the mutex and competes with new arriving goroutines over
	// the ownership. New arriving goroutines have an advantage -- they are
	// already running on CPU and there can be lots of them, so a woken up
	// waiter has good chances of losing. In such case it is queued at front
	// of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
	// it switches mutex to the starvation mode.
	//
	// In starvation mode ownership of the mutex is directly handed off from
	// the unlocking goroutine to the waiter at the front of the queue.
	// New arriving goroutines don't try to acquire the mutex even if it appears
	// to be unlocked, and don't try to spin. Instead they queue themselves at
	// the tail of the wait queue.
	//
	// If a waiter receives ownership of the mutex and sees that either
	// (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
	// it switches mutex back to normal operation mode.
	//
	// Normal mode has considerably better performance as a goroutine can acquire
	// a mutex several times in a row even if there are blocked waiters.
	// Starvation mode is important to prevent pathological cases of tail latency.
	starvationThresholdNs = 1e6
)
```

其实上面的注释写的比较明白:

- `正常模式`: `mutex`的等待者都以FIFO顺序在排队，前一个协程执行结束解锁时，会唤醒等待队列中的一个协程，唤醒的协程和刚来的协程（**还没进入队列**）竞争锁的所有权,
	因为刚进来的协程当前正在持有着cpu，资源也不需要重新调度，对比之下刚唤醒的协程大概率竞争不过新来的协程，所以当唤醒的协程沉睡超过`1ms`，会将锁置为饥饿模式;

- `饥饿模式`: 解锁的协程将锁的所有权移交给等待队列中的**队首**协程，新到来的协程**不会**尝试去获取锁的控制权（即时当前是可获取状态），也**不会**尝试去自旋，会直接加入到队尾等待被唤醒;

`饥饿模式`还会解除:
- 当前获取`mutex`所有权的协程是阻塞队列的最后一个协程;
- 或者该协程等待时间小于`1ms`，则将饥饿模式转换成正常模式;