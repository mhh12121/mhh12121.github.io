---
title: Golang timer
date: 2020-02-03 14:20
tags: golang
---
//TODO
<!--more-->

我们比较熟悉的时间包：
## 定义

1. timer.C 是 一个channel，在timer过期后，这个只读chan会有一个值
2. 除了AfterFunc方法外，一个timer一定要由NewTimer创建


```go
// The Timer type represents a single event.
// When the Timer expires, the current time will be sent on C,
// unless the Timer was created by AfterFunc.
// A Timer must be created with NewTimer or AfterFunc.

type Timer struct {
	C <-chan Time //一个channel，在timer过期后，这个只读chan会有一个值
	r runtimeTimer
}
```

在1.13以下(实际还要加上1.10以后),timer的实现有些不一样，首先timer结构复杂了一些:

`timerproc`和小顶堆分成最多64个`timerproc`协程和四叉堆，
用来休眠就近时间的方法还是依赖`futex timeout`机制。默认timerproc数量会跟`GOMAXPROCS`一致的，但最大也就64个，因为会被64取摸;


但在1.14下，其性能优化了了几个数量级:

- 其将存放事件的四叉堆放到了P中
- 取消了`timerproc`，使用netpoll的epollwait来做就近时间的休眠等待(这样每次`runtime.schedule`都可以检查到定时器上有无运行到时的timer)；

### 1.13版本下
回顾一下:
- new一个`timer`使用的是`NewTimer()`函数，实际调用了`startTimer(t *timer)`,
- 而`startTimer`函数实际就是调用了`runtime.addTimer`函数，实际增加一个timer t到当前`P`上，避免了修改其他P的堆上的timer的`when`字段，可能会导致堆无法排序:

timer的结构是一个64长度的数组:

```go
const timersLen = 64

var timers [timersLen]struct {
    timersBucket
    // The padding should eliminate false sharing
	// between timersBucket values.
	pad [cpu.CacheLinePadSize - unsafe.Sizeof(timersBucket{})%cpu.CacheLinePadSize]byte
}
//runtime下
//go:notinheap
type timersBucket struct {
	lock         mutex
	gp           *g
	created      bool
	sleeping     bool
	rescheduling bool
	sleepUntil   int64
	waitnote     note
	t            []*timer
}

// Package time knows the layout of this structure.
// If this struct changes, adjust ../time/sleep.go:/runtimeTimer.
// For GOOS=nacl, package syscall knows the layout of this structure.
// If this struct changes, adjust ../syscall/net_nacl.go:/runtimeTimer.
type timer struct {
	tb *timersBucket // the bucket the timer lives in
	i  int           // heap index

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr
}

```

runtimeTimer结构: 
是一个接口

```go
// Interface to timers implemented in package runtime.
// Must be in sync with ../runtime/time.go:/^type timer
//同runtime.timer结构一样
type runtimeTimer struct {
	tb uintptr
	i  int

	when   int64
	period int64 //是否是周期运行
	f      func(interface{}, uintptr) // NOTE: must not be closure,不能是闭包？？？，初始化计时器
	arg    interface{}
	seq    uintptr
}


```

调用的函数:

```go
//1.13
func addtimer(t *timer) {
    // 得到要被插入的 bucket
    tb := t.assignBucket()

    // 加锁，将timer插入到bucket中
    lock(&tb.lock)
    ok := tb.addtimerLocked(t)
    unlock(&tb.lock)

    if !ok {
    badTimer()
    }
}
//分配timer buckets
func (t *timer) assignBucket() *timersBucket {
    id := uint8(getg().m.p.ptr().id) % timersLen
    t.tb = &timers[id].timersBucket
    return t.tb
}
func (tb *timersBucket) addtimerLocked(t *timer) bool {
    t.i = len(tb.t)
    tb.t = append(tb.t, t)
    if !siftupTimer(tb.t, t.i) {
        return false
    }
    if t.i == 0 {
        if tb.sleeping && tb.sleepUntil > t.when {
            tb.sleeping = false
            notewakeup(&tb.waitnote)
        }
        ...
        if !tb.created {
            tb.created = true
            go timerproc(tb)
        }
    }
    return true
}
```
我们发现有一个bucket的概念:

在1.13之前，有64个全局的timerBucket，timer整个生命周期全部由timerBucket管理和调度;

- 然后再调用`timerproc`方法从堆顶拿timer，判断是否过期，到期就执行

- bucket中无任务时，会调用`goparkunlock`来休眠该goroutine

- 至少有一个timer任务时，`notetsleepg`传入下次到期时间来休眠（`notetsleepg`其实有调用`entrysyscallback`触发`handoffp`，即一定触发到调度）

```go
func timerproc(tb *timersBucket) {
    tb.gp = getg()
	for {
		lock(&tb.lock)
		tb.sleeping = false
		now := nanotime()
		delta := int64(-1)
		for {
			if len(tb.t) == 0 {
				delta = -1
				break
			}
			t := tb.t[0]
            delta = t.when - now
            //timer未到期
			if delta > 0 {
				break
			}
			ok := true
			if t.period > 0 {
				// leave in heap but adjust next time to fire
				t.when += t.period * (1 + -delta/t.period)
				if !siftdownTimer(tb.t, 0) {
					ok = false
				}
			} else {
				// remove from heap
				last := len(tb.t) - 1
				if last > 0 {
					tb.t[0] = tb.t[last]
					tb.t[0].i = 0
				}
				tb.t[last] = nil
				tb.t = tb.t[:last]
				if last > 0 {
					if !siftdownTimer(tb.t, 0) {
						ok = false
					}
				}
				t.i = -1 // mark as removed
			}
			f := t.f
			arg := t.arg
			seq := t.seq
			unlock(&tb.lock)
			if !ok {
				badTimer()
			}
			...
			f(arg, seq)
			lock(&tb.lock)
        }
        //无任务剩下
		if delta < 0 || faketime > 0 {
			// No timers left - put goroutine to sleep.
			tb.rescheduling = true
			goparkunlock(&tb.lock, waitReasonTimerGoroutineIdle, traceEvGoBlock, 1)
			continue
		}
		// At least one timer pending. Sleep until then.
		tb.sleeping = true
		tb.sleepUntil = now + delta
		noteclear(&tb.waitnote)
		unlock(&tb.lock)
		notetsleepg(&tb.waitnote, delta)
	}
}
```
- 其中`notetsleepg`会调用`notetsleepg_internal`,该函数最后实际会调用`futexsleep`来休眠
- 相应的要使用`futexwakeup`来唤醒


```go
// Atomically,
//	if(*addr == val) sleep
// Might be woken up spuriously; that's allowed.
// Don't sleep longer than ns; ns < 0 means forever.
//go:nosplit
func futexsleep(addr *uint32, val uint32, ns int64) {
	// Some Linux kernels have a bug where futex of
	// FUTEX_WAIT returns an internal error code
	// as an errno. Libpthread ignores the return value
	// here, and so can we: as it says a few lines up,
	// spurious wakeups are allowed.
	if ns < 0 {
		futex(unsafe.Pointer(addr), _FUTEX_WAIT_PRIVATE, val, nil, nil, 0)
		return
	}

	var ts timespec
	ts.setNsec(ns)
	futex(unsafe.Pointer(addr), _FUTEX_WAIT_PRIVATE, val, unsafe.Pointer(&ts), nil, 0)
}
// If any procs are sleeping on addr, wake up at most cnt.
//go:nosplit
func futexwakeup(addr *uint32, cnt uint32) {
	ret := futex(unsafe.Pointer(addr), _FUTEX_WAKE_PRIVATE, cnt, nil, nil, 0)
	if ret >= 0 {
		return
	}

	// I don't know that futex wakeup can return
	// EAGAIN or EINTR, but if it does, it would be
	// safe to loop and call futex again.
	systemstack(func() {
		print("futexwakeup addr=", addr, " returned ", ret, "\n")
	})

	*(*int32)(unsafe.Pointer(uintptr(0x1006))) = 0x1006
}
```
### 1.14版本下:

处理器P中有:`timers`数组，用作四叉堆:
timer的结构:
- `pp`:
- `when`和`period`: 会在`when`的时候唤醒，然后下一次`when`+`period`
- `f`:timer调用的function应该well-behave(????不是很明白)且不能阻塞的


```go

type p struct {
        // 保护timers堆读写安全
        timersLock mutex

        // 存放定时器任务
        timers []*timer
    
        ,,,
}
//可以对比一下1.13的timer
//runtime.timer
type timer struct {
    // If this timer is on a heap, which P's heap it is on.
	// puintptr rather than *p to match uintptr in the versions
	// of this struct defined in other packages.
    pp puintptr  // p的位置
    // Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
    when   int64 // 到期时间
    period int64 // 周期时间，适合ticker
    f      func(interface{}, uintptr) // 回调方法
    arg    interface{}  // 参数
    seq    uintptr  // 序号

    //// What to set the when field to in timerModifiedXX status.
    nextwhen int64 // 下次的到期时间
    status uint32 // 状态
}
```

- `status`:取值如下所示:

```go
// Values for the timer status field.
const (
	// Timer has no status set yet.
	timerNoStatus = iota

	// Waiting for timer to fire.
	// The timer is in some P's heap.
	timerWaiting

	// Running the timer function.
	// A timer will only have this status briefly.
	timerRunning

	// The timer is deleted and should be removed.
	// It should not be run, but it is still in some P's heap.
	timerDeleted

	// The timer is being removed.
	// The timer will only have this status briefly.
	timerRemoving

	// The timer has been stopped.
	// It is not in any P's heap.
	timerRemoved

	// The timer is being modified.
	// The timer will only have this status briefly.
	timerModifying

	// The timer has been modified to an earlier time.
	// The new when value is in the nextwhen field.
	// The timer is in some P's heap, possibly in the wrong place.
	timerModifiedEarlier

	// The timer has been modified to the same or a later time.
	// The new when value is in the nextwhen field.
	// The timer is in some P's heap, possibly in the wrong place.
	timerModifiedLater

	// The timer has been modified and is being moved.
	// The timer will only have this status briefly.
	timerMoving
)
```

#### 大概流程

- `startTimer`：`time/sleep.go`里面的startTimer实际就是runtime里面的startTimer 
- `addTimer`:将定时任务放到当前P中
-  `wakeNetPoller(when)`: when之前不会wakeup任何一个，全局的`sched.lastpoll`记录到上一次是否有wakeup，如果无就进行判断,


```go
// startTimer adds t to the timer heap.
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
	if raceenabled {
		racerelease(unsafe.Pointer(t))
	}
	addtimer(t)
}
// addtimer adds a timer to the current P.
// This should only be called with a newly created timer.
// That avoids the risk of changing the when field of a timer in some P's heap,
// which could cause the heap to become unsorted.
func addtimer(t *timer) {
	// when must never be negative; otherwise runtimer will overflow
	// during its delta calculation and never expire other runtime timers.
	if t.when < 0 {
		t.when = maxWhen
	}
	if t.status != timerNoStatus {
		throw("addtimer called with initialized timer")
	}
	t.status = timerWaiting

	when := t.when

	pp := getg().m.p.ptr()
	lock(&pp.timersLock)
	cleantimers(pp)
	doaddtimer(pp, t)
	unlock(&pp.timersLock)

	wakeNetPoller(when)
}
// wakeNetPoller wakes up the thread sleeping in the network poller,
// if there is one, and if it isn't going to wake up anyhow before
// the when argument.
func wakeNetPoller(when int64) {
	if atomic.Load64(&sched.lastpoll) == 0 {
		// In findrunnable we ensure that when polling the pollUntil
		// field is either zero or the time to which the current
		// poll is expected to run. This can have a spurious wakeup
		// but should never miss a wakeup.
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > when {
			netpollBreak()
		}
	}
}
```
这里注意`wakeNetPoller`激活netpoll的等待，是在`findrunnable`方法里面会超时阻塞(再次复习一次):

- 初始化`netpollinit`中， 会创建一个读(netpollBreakRd)，写(netpollBreakWr)的管道(通过`nonblockingPipe`)
- 而激活的`wakeNetPoller`实际就是往`netpollBreakWr`管道里面写入东西，自然就会唤醒netPoll


```go
var (
	epfd int32 = -1 // epoll descriptor

	netpollBreakRd, netpollBreakWr uintptr // for netpollBreak
)

func netpollinit() {
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
	r, w, errno := nonblockingPipe()
	if errno != 0 {
		println("runtime: pipe failed with", -errno)
		throw("runtime: pipe failed")
	}
	ev := epollevent{
		events: _EPOLLIN,
	}
	*(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd
	errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
	if errno != 0 {
		println("runtime: epollctl failed with", -errno)
		throw("runtime: epollctl failed")
	}
	netpollBreakRd = uintptr(r)
	netpollBreakWr = uintptr(w)
}

// netpollBreak interrupts an epollwait.
func netpollBreak() {
	for {
		var b byte
		n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
		if n == 1 {
			break
		}
		if n == -_EINTR {
			continue
		}
		if n == -_EAGAIN {
			return
		}
		println("runtime: netpollBreak write failed with", -n)
		throw("runtime: netpollBreak write failed")
	}
}

```

- 而这一切都是在`findrunnable`中尝试进行netpoll检查时`netpoll(0)`方法用到epoll_wait,这里不仅监控到红黑树上的fd,又可以监控到定时任务的等待;
- 注意一下`checkTimers`方法获取`pollUntil`的方法，下面会继续讲解


```go
// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
    ...
    // Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
    // anyway.
    //这里是非阻塞，还没进去呢
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
    }
    ...
    //尝试4次，从其他p的runq偷，再从其他p的timer偷
    for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			p2 := allp[enum.position()]
			if _p_ == p2 {
				continue
			}
			if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
				return gp, false
			}

			// Consider stealing timers from p2.
			// This call to checkTimers is the only place where
			// we hold a lock on a different P's timers.
			// Lock contention can be a problem here, so avoid
			// grabbing the lock if p2 is running and not marked
			// for preemption. If p2 is running and not being
            // preempted we assume it will handle its own timers.
            //p2在运行且不在被抢占中，才可以认为其可以处理自己的timers
			if i > 2 && shouldStealTimers(p2) {
                //执行已经到期的定时任务
                //注意这里返回的w，下面详细讲解...
				tnow, w, ran := checkTimers(p2, now)
                now = tnow
                
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
				if ran {
					// Running the timers may have
					// made an arbitrary number of G's
					// ready and added them to this P's
					// local run queue. That invalidates
					// the assumption of runqsteal
					// that is always has room to add
					// stolen G's. So check now if there
					// is a local G to run.
					if gp, inheritTime := runqget(_p_); gp != nil {
						return gp, inheritTime
					}
					ranTimer = true
				}
			}
		}
	}
    //计算delta值，距离当前时间最近的时间点的时间差
    delta := int64(-1)
	if pollUntil != 0 {
		// checkTimers ensures that polluntil > now.
		delta = pollUntil - now
	}
    ...
    //最后还要进行一次阻塞的pollnetwork
    // poll network
	if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || pollUntil != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		atomic.Store64(&sched.pollUntil, uint64(pollUntil))
		if _g_.m.p != 0 {
			throw("findrunnable: netpoll with p")
		}
		if _g_.m.spinning {
			throw("findrunnable: netpoll with spinning")
		}
		if faketime != 0 {
			// When using fake time, just poll.
			delta = 0
		}
		list := netpoll(delta) // block until new work is available
		atomic.Store64(&sched.pollUntil, 0)
		atomic.Store64(&sched.lastpoll, uint64(nanotime()))
		if faketime != 0 && list.empty() {
			// Using fake time and nothing is ready; stop M.
			// When all M's stop, checkdead will call timejump.
			stopm()
			goto top
		}
}
// netpoll checks for ready network connections.
// Returns list of goroutines that become runnable.
// delay < 0: blocks indefinitely
// delay == 0: does not block, just polls
// delay > 0: block for up to that many nanoseconds
func netpoll(delay int64) gList {
    ...
retry:
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    ...
}
```

#### CheckTimers方法

继续上面的`checkTimers`方法:
该方法主要是检查传入P上任何准备就绪的timers，通过`runtimers`来运行到期的定时任务，返回`下一次到期时间`以及`是否有定时任务到期`



```go
// checkTimers runs any timers for the P that are ready.
// If now is not 0 it is the current time.
// It returns the current time or 0 if it is not known,
// and the time when the next timer should run or 0 if there is no next timer,
// and reports whether it ran any timers.
// If the time when the next timer should run is not 0,
// it is always larger than the returned time.
// We pass now in and out to avoid extra calls of nanotime.
//go:yeswritebarrierrec
func checkTimers(pp *p, now int64) (rnow, pollUntil int64, ran bool) {
	// If there are no timers to adjust, and the first timer on
	// the heap is not yet ready to run, then there is nothing to do.
	if atomic.Load(&pp.adjustTimers) == 0 {
		next := int64(atomic.Load64(&pp.timer0When))
		if next == 0 {
			return now, 0, false
		}
		if now == 0 {
			now = nanotime()
		}
		if now < next {
			// Next timer is not ready to run.
			// But keep going if we would clear deleted timers.
			// This corresponds to the condition below where
			// we decide whether to call clearDeletedTimers.
			if pp != getg().m.p.ptr() || int(atomic.Load(&pp.deletedTimers)) <= int(atomic.Load(&pp.numTimers)/4) {
				return now, next, false
			}
		}
	}

	lock(&pp.timersLock)

	adjusttimers(pp)

	rnow = now
	if len(pp.timers) > 0 {
		if rnow == 0 {
			rnow = nanotime()
		}
		for len(pp.timers) > 0 {
			// Note that runtimer may temporarily unlock
            // pp.timersLock.
            //运行定时任务
			if tw := runtimer(pp, rnow); tw != 0 {
				if tw > 0 {
					pollUntil = tw
				}
				break
			}
			ran = true
		}
	}

	// If this is the local P, and there are a lot of deleted timers,
	// clear them out. We only do this for the local P to reduce
	// lock contention on timersLock.
	if pp == getg().m.p.ptr() && int(atomic.Load(&pp.deletedTimers)) > len(pp.timers)/4 {
		clearDeletedTimers(pp)
	}

	unlock(&pp.timersLock)

	return rnow, pollUntil, ran
}
```
其中`runtimers`方法:
- 遍历p的timer堆顶任务是否到期，如果是`timerWaiting`状态，进入尝试执行
- `runOneTimer`尝试执行，如果是周期任务，还会重新入队;

```go
// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.
//go:systemstack
func runtimer(pp *p, now int64) int64 {
	for {
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("runtimer: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerWaiting:
			if t.when > now {
				// Not ready to run.
				return t.when
			}

			if !atomic.Cas(&t.status, s, timerRunning) {
				continue
			}
			// Note that runOneTimer may temporarily unlock
			// pp.timersLock.
			runOneTimer(pp, t, now)
			return 0

		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
			if len(pp.timers) == 0 {
				return -1
			}

		case timerModifiedEarlier, timerModifiedLater:
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			t.when = t.nextwhen
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}

		case timerModifying:
			// Wait for modification to complete.
			osyield()

		case timerNoStatus, timerRemoved:
			// Should not see a new or inactive timer on the heap.
			badTimer()
		case timerRunning, timerRemoving, timerMoving:
			// These should only be set when timers are locked,
			// and we didn't do it.
			badTimer()
		default:
			badTimer()
		}
	}
}

// runOneTimer runs a single timer.
// The caller must have locked the timers for pp.
// This will temporarily unlock the timers while running the timer function.
//go:systemstack
func runOneTimer(pp *p, t *timer, now int64) {
	...

	f := t.f
	arg := t.arg
	seq := t.seq

	if t.period > 0 {
		// Leave in heap but adjust next time to fire.
		delta := t.when - now
		t.when += t.period * (1 + -delta/t.period)
		siftdownTimer(pp.timers, 0)
		if !atomic.Cas(&t.status, timerRunning, timerWaiting) {
			badTimer()
		}
		updateTimer0When(pp)
	} else {
		// Remove from heap.
		dodeltimer0(pp)
		if !atomic.Cas(&t.status, timerRunning, timerNoStatus) {
			badTimer()
		}
	}

	...

	unlock(&pp.timersLock)

	f(arg, seq)

	lock(&pp.timersLock)
    ...
}
```

#### Reset(d Duration)方法

1. 返回true，如果这个timer被激活；返回false如果这个timer被停止或者过期(返回值最主要用来保持兼容性)
2. Reset只能被 停止或者过期的带有空(队列已经为空)channels的timers 调用
3. 


```go
// Reset changes the timer to expire after duration d.
// It returns true if the timer had been active, false if the timer had
// expired or been stopped.
//
// Reset should be invoked only on stopped or expired timers with drained channels.
// If a program has already received a value from t.C, the timer is known
// to have expired and the channel drained, so t.Reset can be used directly.
// If a program has not yet received a value from t.C, however,
// the timer must be stopped and—if Stop reports that the timer expired
// before being stopped—the channel explicitly drained:
//
// 	if !t.Stop() {
// 		<-t.C
// 	}
// 	t.Reset(d)
//
// This should not be done concurrent to other receives from the Timer's
// channel.
//
// Note that it is not possible to use Reset's return value correctly, as there
// is a race condition between draining the channel and the new timer expiring.
// Reset should always be invoked on stopped or expired channels, as described above.
// The return value exists to preserve compatibility with existing programs.
func (t *Timer) Reset(d Duration) bool {
	if t.r.f == nil {
		panic("time: Reset called on uninitialized Timer")
	}
	w := when(d)
	active := stopTimer(&t.r)
	t.r.when = w
	startTimer(&t.r)
	return active
}
```



## 性能分析

1. 锁竞争?

    go1.14虽然将timer放在timer内，但是p操作heap也是需要锁的(`findrunnable`可能要偷其他p的timers)；
    go1.13`timerProcs`有timers和对应的锁,最大64个,但是最大的问题是`timerproc`操作`notetsleepg`会有系统调用，接着`handoffp`，这里要涉及全局`sched.lock`锁;
    
2. runtime调度次数

    go1.13的`timerproc`本身就是属于goroutine，受到runtime的调度影响；
    go1.14则将timer的工作交给了`runtime.schedule`，不需要额外调度;

3. 上下文切换
    新增任务时，futex(在竞争导致操作结果不一致的时候还是会进入kernel)和epoll_wait都是syscall，无区别；
    但是go1.14无`timerproc`,新任务可以直接插入或者多次插入后再考虑是否休眠;



## 引出
### Race Detector(竞态条件检测)

很简单:
```go
go run -race main.go 
go build -race mycmd   // build the command
go install -race mypkg // install the package
```

```go  
//官网的例子：
func main() {
     start := time.Now()
    var t *time.Timer
     t = time.AfterFunc(randomDuration(), func() {//这里的t是写入
         fmt.Println(time.Now().Sub(start))
         t.Reset(randomDuration()) //这里的t是读取，可能在一定情况下，randomDuration使得在这里读取前将t置为nil，所以读取了空值，会报nil pointer错误
     })
     time.Sleep(5 * time.Second)
 }
  func randomDuration() time.Duration {
     return time.Duration(rand.Int63n(1e9))
 }
```


改成以下版本:
主要是让t变量只能从main的goroutine中读取和写入

```go
 func main() {
     start := time.Now()
     reset := make(chan bool)
     var t *time.Timer
     t = time.AfterFunc(randomDuration(), func() {
         fmt.Println(time.Now().Sub(start))
         reset <- true
     })
     for time.Since(start) < 5*time.Second {
         <-reset
         t.Reset(randomDuration())
     }
 }

```
