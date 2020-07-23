---
title: Goroutine Notes
date: 2019-07-09 01:00
tags: Golang
---

Goroutine 的模型，调度等，它与普通thread有何区别？先留个坑//todo
<!--more-->
## 为什么有这个东西？
1. 传统OS自带的线程一个占栈1MB，明显大的过分，所以编程语言自身得另外实现一些小的线程, 而goroutines一般就4KB左右，当然这个数值是可以调整的；

2. 切换上下文的时候一般一个线程就消耗1μs,但是goroutine的切换则仅仅有0.2μs左右，大约快了80%;（里面避免了内核和用户态上的切换）


###  引用
《 Scalable Go Scheduler Design Doc》中有描述
> Goroutines are part of making concurrency easy to use. The idea, which has been around for a while, is to multiplex independently    executing functions—coroutines—onto a set of threads. When a coroutine blocks, such as by calling a blocking system call, the run-time automatically moves other coroutines on the same operating system thread to a different, runnable thread so they won't be blocked. The programmer sees none of this, which is the point. The result, which we call goroutines, can be very cheap: unless they spend a lot of time in long-running system calls, they cost little more than the memory for the stack, which is just a few kilobytes.

大概意思就是 当系统调用阻塞，runtime环境会自动把被阻塞在当前线程内的coroutines移到另一个线程，这种在go里面就叫goroutines;

而针对goroutines的大小，也做了如下设计:
> To make the stacks small, Go's run-time uses segmented stacks. A newly minted goroutine is given a few kilobytes, which is almost always enough. When it isn't, the run-time allocates (and frees) extension segments automatically. The overhead averages about three cheap instructions per function call. It is practical to create hundreds of thousands of goroutines in the same address space. If goroutines were just threads, system resources would run out at a much smaller number.

然后对于goroutine的栈设计，使用了**分段**的栈， 而且对于分段的栈增加了灵活性，当空间不足的话就会自动分配更多的空间，而且因为这个不涉及内核层面，不用保存过多信息，所以你可以在同一个地址空间里创建上千个goroutines

- 这些分段栈的基本功能

    1. 保护回复上下文的函数
    2. 运行队列processQueue

要时刻明白对于线程来讲，其**阻塞指的是切换了调度队列**，不再进行当前的**数据控制流**，如果其他流满足条件，则会移出当前队列，调度会之前的数据流。同理goroutine也只是一个结构，记录了运行的函数，运行的位置等


## 调度模型

一般来说多线程调度模型有 work-sharing 和 work-stealing模型

go采用了后者，可以看看有关[work-stealing的论文](http://supertech.csail.mit.edu/papers/steal.pdf)

架构：
GPM
- G（goroutine）指的是go语言的goroutine（有些人叫它为协程，但其实跟coroutine有一点区别，因为coroutine单纯在用户态使用）

### 调度时机

1. Channel,mutex之类同步操作发生阻塞
2. time.sleep
3. 主动调用runtime.GoSched()
4. 网络IO阻塞
5. gc
6. 运行过久或者系统调用过久


### Scheduler调度过程

#### 大概流程
调度前的检查:
1. 是否分配有gc mark，如果有则要做gc mark
2. 检查有无localq，有就运行
3. 没有则看globalq
4. 看一下net中有无poll的出来
5. 从其他的p 偷一部分

#### 相关状态转换

[一个状态图](/img/gstatus.png)

当有新的Goroutine被创建或者是现存的goroutine更新为runnable状态，它会被push到当前P的runnable goroutine list里面，
当P完成了执行goroutine，它会
- 首先从自己的runnable g list里面pop一个goroutine，如果list是空的，它会随机选取其他P，并且偷取其list的一半runnable goroutine

当M 创建了新的goroutine，它要保证有其他M执行这个goroutine
同样的，如果M进入了syscall阶段，它也要保证有其他M可以执行这个goroutine

禁止抢占:
```go
func runtime_procPin() int //标记当前G在M上不会被抢占，并返回当前P的ID
```
```go
func runtime_procUnpin() //解除抢占标志

```
这里我们有两种方法：
我们可以立即block或unblock多个M，或使其自旋；
但这里会有性能损耗和花费不必要的cpu周期，方法是使用自旋而且burn CPU cycles
然而，这不应该影响在GOMAXPROCS=1的程序（command line，appengine这些）

自旋有两个等级：
1. 一个已经附着了一个P的idle M 会不断自旋寻找新的Goroutines
2. 一个已经附着了一个P的M 自旋等待其他可用的P 
以上中，最多有GOMAXPROCS个自旋的goroutines， 等级（1）的idle M 不会阻塞 即使有等级（2）的idle M；
当新的goroutine被创建或者M进入syscall或者M从idle变成busy，它会保证至少存在一个自旋M （或者所有P是busy），
1. 这就保证了不会有当前运行着的Goroutines被其他 M 运行
2. 也避免了过多的 M 同时 阻塞和释放阻塞

### Sysmon

sysmon是在**runtime初始化之后，执行代码之前**，由runtime启动且不与任何P绑定直接由一个M执行的协程，类似于linux的一些系统任务内核线程

具体设置如![sysmon状态转换图](img/sysmon.png)


### 各个结构体
- g (goroutine) 
```go
type g struct {
	goid           int64
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	//stackguard0用作栈的指针
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	//栈空间[lo,hi)
	//type stack struct {
	//	lo uintptr
	//	hi uintptr
	//}
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink
    ...
	m              *m      // current m; offset known to arm liblink
	//调度器,上下文保存的信息所在地
	sched          gobuf
    ...
	param          unsafe.Pointer // passed parameter on wakeup
	...
	
	schedlink      guintptr
	waitsince      int64      // approx time when the g become blocked
	waitreason     waitReason // if status==Gwaiting
	preempt        bool       // preemption signal, duplicates stackguard0 = stackpreempt
	...
	preemptscan    bool       // preempted g does scan for gc
	gcscandone     bool       // g has scanned stack; protected by _Gscan bit in status
	gcscanvalid    bool       // false at start of gc cycle, true if G has not run since last scan; TODO: remove?
	...
	raceignore     int8       // ignore race detection events
	sysblocktraced bool       // StartTrace has emitted EvGoInSyscall about this goroutine
	sysexitticks   int64      // cputicks when syscall has returned (for tracing)
	traceseq       uint64     // trace event sequencer
	tracelastp     puintptr   // last P emitted an event for this goroutine
    lockedm        muintptr
    
    //本身的寄存器状态
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
    gopc           uintptr         // pc of go statement that created this goroutine
    ....
	startpc        uintptr         // pc of goroutine function
	racectx        uintptr
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order

	.....
	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}

```


- M（machine）指的就是OS原生线程，是真正调度资源的单位，M是idle或者syscall中，需要P的调度
```go
type m struct {
	id            int64
    //g0是一个调用栈
	g0      *g     // goroutine with scheduling stack
	morebuf gobuf  // gobuf arg to morestack
	procid        uint64       // for debuggers, but offset not hard-coded
	//底层的线程id
    ...
    //这个信号处理的goroutines
	gsignal       *g           // signal-handling g
	goSigStack    gsignalStack // Go-allocated signal handling stack
    sigmask       sigset       // storage for saved signal mask
	//TLS启动时候要使用
	//传给FS寄存器的局部变量
	tls           [6]uintptr   // thread-local storage (for x86 extern register)
	//m启动时的函数，会传给clone
	mstartfn      func(){}  //当前运行的goroutine,{}在语法中是错误的，这里为了使markdown解析而加上
	//当前运行代码的g
    curg          *g       // current running goroutine
    
	caughtsig     guintptr // goroutine running during fatal signal
    p             puintptr // attached p for executing go code (nil if not executing go code)
    
	nextp         puintptr
    oldp          puintptr // the p that was attached before executing a syscall

	mallocing     int32
    throwing      int32
    //如果不等于"",没有发生抢占
    preemptoff    string // if != "", keep curg running on this m
    
	locks         int32
	....
    
    //m正在自旋，寻找可以attach的工作对象(P), m找不到可运行的g
	spinning      bool // m is out of work and is actively looking for work
	blocked       bool // m is blocked on a note
    .....
    //如果=0，则可以清空g0以及清除该m，是原子性的操作
	freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
	fastrand      [2]uint32
	needextram    bool
    traceback     uint8
	...//里面一些cgo的代码
	park          note
    alllink       *m // on allm
    //调度链，是一个m的指针
    schedlink     muintptr
	//每一个P(Per-thread)的用于存储小对象的cache,没有锁，因为都在一个P内，运行代码时绑定的p中的mcache
    mcache        *mcache
    
	//goroutine的指针,uintptr可以避过写屏障, 主要用于Gobuf goroutine状态或者是那些不经过P的调度列表
	//是否与某个g一直绑定
    lockedg       guintptr
    //创建当前thread的栈
	createstack   [32]uintptr // stack that created this thread.
    
    ...//track用
    //下一个等待锁的M
	nextwaitm     muintptr    // next m waiting for lock
    //一些锁的操作
    waitunlockf   func(*g, unsafe.Pointer) bool
	waitlock      unsafe.Pointer
	waittraceev   byte
	waittraceskip int
	startingtrace bool
	syscalltick   uint32
	thread        uintptr // thread handle
	freelink      *m      // on sched.freem
    ...
    //debug
	dlogPerM
    //表明操作系统相关
	mOS
}

```


- P（Process）指的是go语言中的调度器，M就是用P才能调度G
可以看到P是内嵌于 M 和 G 之间的，
```go
type p struct {
    //每一个p都有自己的id
    id          int32
    //状态，有_Pidle ,_Prunning,_Psyscall, _Pgcstop, _Pdead
    status      uint32 // one of pidle/prunning/...
    
    link        puintptr
    
    //每次调度都会自增
	schedtick   uint32     // incremented on every scheduler call
    syscalltick uint32     // incremented on every system call
    //go程序启动时候的sysmon用
    sysmontick  sysmontick // last tick observed by sysmon
    //指的是后面指针连接的一个m,同时该m也有一个指针连向自己 ???
    m           muintptr   // back-link to associated m (nil if idle)
    //
	mcache      *mcache
	raceprocctx uintptr

    //defer的池,defer函数，结构在此
	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

    //goroutine的id生成，能平均分到每一个idgen中
	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64


    //这个就是连接的可运行的goroutines队列,可以不加锁访问（都在一个P里面，没必要加锁）
	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	//如果runnext非空，则是一个runnable状态的g，如果在当前时间片中还有剩余，则runnext指向的就是下一个应该运行的g而不使用runq里面的g，其会继承剩下的时间；
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}

    //sudog相关
	sudogcache []*sudog
	sudogbuf   [128]*sudog

	...//trace的一些东西

	palloc persistentAlloc // per-P to avoid mutex

    //用作优化内存对齐
	_ uint32 // Alignment for atomic fields below

	// Per-P GC state
	gcAssistTime         int64    // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64    // Nanoseconds in fractional mark worker (atomic)
	gcBgMarkWorker       guintptr // (atomic)
	gcMarkWorkerMode     gcMarkWorkerMode

	// gcMarkWorkerStartTime is the nanotime() at which this mark
	// worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	// wbBuf is this P's GC write barrier buffer.
	//
	// TODO: Consider caching this in the running G.
	wbBuf wbBuf

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	pad cpu.CacheLinePad
}


```

- SchedDt 调度结构
```go


```
相关结构可以在runtime/runtime2.go 中找到

P在GOMAXPROCS中，所有的P被组织成一个数组，当GOMAXPROCS改变时会触发 stop the world来重新调整P 数组的长度
一些变量会从sched中分离出到P中

### 死锁检测和终止
当所有P是idle的时候进行检测（全局idle P的原子计数）

 
旋转态->不旋态的转换中，
可能和创建一个新的goroutine和创建一部分或其他需要unpark的工作线程 的时候发生竞态条件
如果转换和创建都失败，我们就可以以半静态cpu未充分利用结束；
goroutine 准备步骤是：提交一个goroutine去local queue，store-style memory 屏障，检查sched.nmspinning


不旋态->旋转态是： 减少nmspinning，store-style memory 屏障，检查新的work的所有per-P work queue
而且以上都不适用于global run queue



### 协作式抢占

retake() 调用preemptone()将被抢占的G的stackguard0 设为stackPreempt，
被设置标志的G下一次进行函数调用的时候，检查栈空间失败。然后会触发morestack() (汇编代码,asm_xxx.s)
然后进行一连串的函数调用
大概流程

morestack()--> newstack()--> gopreempt_m() --> goschedImpl() --> schedule()




## 补充：

网上的经验：

这个goroutine类似于线程池管理(c++线程池原理相似)，

1. 遇到阻塞的情况，怎么扩展进程池，使其不会因为任务阻塞或者同步独占线程


1. goroutine类似green threads（Green threads），是application自己维护的执行过程；很多goroutines实际上被有限个操作系统管理的threads执行;
2. goroutine的调度往往发生在I/O和系统调用的时候。如果创建的goroutines都是跑for循环做纯计算（没有I/O），那就需要我们自己时不常的调用 runtime.Gosched()，否则那几个在thread上跑的goroutines会霸占着threads，不让其他goroutines有机会跑起来;

3. 用户代码造成的协程同步造成的阻塞，只是切换(gopark)协程，而不是阻塞线程，**m和p仍结合**，去寻找新的可执行的g;

4. 上层封装了epoll，网络fd会设置成NonBlocking模式，返回EAGAIN则gopark当前goroutine，在m调度，sysmon中，gc start the world等阶段均会poll出ready的goroutine进行运行或者添加到全局runq中


## SystemStack

SystemStack(fn func())
系统栈 被不同地方调用会有不同的表现方式：
- 直接调用fn并返回 需要满足：
    - 被 单个线程的g0 stack调用    或

    -  被信号处理的栈(gsignal)调用,m中有个gsinal字段 ???
- 否则，都从一个普通的goroutine的有限的stack中调用

    表现： 会先切去线程的栈，调用fn，然后切回来该goroutine的栈



```golang
// systemstack runs fn on a system stack.
// If systemstack is called from the per-OS-thread (g0) stack, or
// if systemstack is called from the signal handling (gsignal) stack,
// systemstack calls fn directly and returns.
// Otherwise, systemstack is being called from the limited stack
// of an ordinary goroutine. In this case, systemstack switches
// to the per-OS-thread stack, calls fn, and switches back.
// It is common to use a func literal as the argument, in order
// to share inputs and outputs with the code around the call
// to system stack:
//
//	... set up y ...
//	systemstack(func() {
//		x = bigcall(y)
//	})
//	... use x ...
//
//go:noescape
func systemstack(fn func())
```

### 一个大概的go程序启动流程

golang注释中有大概写明:
```go
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
// The new G calls runtime·main.
```




go程序的入口点是runtime.rt0_go, 流程是:

1. 分配栈空间, 需要2个本地变量+2个函数参数, 然后向8对齐

把传入的argc和argv保存到栈上(rdx寄存器通常用作上下文存储)

更新g0中的stackguard的值, stackguard用于检测栈空间是否不足, 需要分配新的栈空间(栈扩展会申请多一块栈空间并把现在的复制过去)

获取当前cpu的信息并保存到各个全局变量

调用_cgo_init如果函数存在

2. 初始化当前线程的TLS(thread-local-storage), 设置FS寄存器为m0.tls+8(获取时会-8) 
这里跟SP寄存器有关(伪的SP寄存器的地址 = 硬件SP寄存器+8，64位机)

测试TLS是否工作

设置g0到TLS中, 表示当前的g是g0

设置m0.g0 = g0

设置g0.m = m0

3. 调用runtime.check做一些检查

调用runtime.args保存传入的argc和argv到全局变量

调用runtime.osinit根据系统执行不同的初始化

这里(linux x64)设置了全局变量ncpu等于cpu核心数量

4. 调用runtime.schedinit执行共同的初始化

这里的处理比较多, 
- 首先会调用raceinit()检查race condition
- 会初始化栈空间分配器(stackinit), 
- mallocinit()
- mcommoninit(_g_.m),这里是一些公共初始化
- 按cpu核心数量或GOMAXPROCS的值生成P(cpuinit)
- alginit

生成P的处理在procresize中

5. 调用runtime.newproc创建一个新的goroutine, 指向的是runtime.main
runtime.newproc这个函数在创建普通的goroutine时也会使用;

6. 调用runtime·mstart启动m0

- 启动后m0会不断从运行队列获取G并运行, runtime.mstart调用后不会返回
- runtime.mstart这个函数是m的入口点(不仅仅是m0), 在下面的"调度器的实现"中会详细讲解






### runtime.main之后
第一个被调度的G会运行runtime.main, 流程是:

标记主函数已调用, 设置mainStarted = true

启动一个新的M执行sysmon函数, 这个函数会监控全局的状态并对运行时间过长的G进行抢占

要求G必须在当前M(系统主线程)上执行

调用runtime_init函数

调用gcenable函数

调用main.init函数, 如果函数存在

不再要求G必须在当前M上运行

如果程序是作为c的类库编译的, 在这里返回

调用main.main函数

如果当前发生了panic, 则等待panic处理

调用exit(0)退出程序


## Defer函数

平常用的
```go
func do(){
	defer done()
}

```
其结构在 runtime2.go 结构体g中

```go
type g struct{
	goid int64
	...
	//其结构有些在stack中有些在heap中，但是逻辑上都属于stack，所以写屏障是没有必要的;
	_defer *defer{
		siz     int32 // includes both arguments and results
		started bool
		heap    bool
		sp      uintptr // sp at time of defer
		pc      uintptr
		fn      *funcval //调用的函数
		_panic  *_panic // panic that is running defer
		link    *_defer
	} 
	...

}

```